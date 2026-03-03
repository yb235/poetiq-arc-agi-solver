# System Architecture

This document explains *what* the Poetiq ARC-AGI Solver is made of and *how* the pieces fit together. Think of it as the blueprint of the house before you start walking around the rooms.

---

## 1. Bird's-Eye View

```
┌────────────────────────────────────────────────────────────────┐
│                         main.py                                │
│  (entry point – loads data, drives the async task loop,        │
│   writes the Kaggle submission JSON)                           │
└────────────────────────┬───────────────────────────────────────┘
                         │ calls
                         ▼
┌────────────────────────────────────────────────────────────────┐
│                   arc_agi/solve.py                             │
│  (thin dispatch layer – reads CONFIG_LIST and hands off to     │
│   the parallel solver)                                         │
└────────────────────────┬───────────────────────────────────────┘
                         │ calls
                         ▼
┌────────────────────────────────────────────────────────────────┐
│            arc_agi/solve_parallel_coding.py                    │
│  (orchestrator – launches N experts concurrently, then         │
│   aggregates their results via voting)                         │
└───────────┬──────────────────────┬─────────────────────────────┘
            │ spawns N tasks       │ aggregates results
            ▼                      ▼
┌───────────────────────┐  ┌──────────────────────────────────────┐
│ arc_agi/solve_coding  │  │  Voting / Ranking logic              │
│ .py  (one "expert")   │  │  (inside solve_parallel_coding.py)   │
│                       │  │                                      │
│ Inner loop:           │  │  - Group by identical test output    │
│  build prompt         │  │  - Promote largest group (most       │
│  → call LLM           │  │    "votes") to the top               │
│  → parse Python code  │  │  - Break ties by soft score          │
│  → run in sandbox     │  └──────────────────────────────────────┘
│  → evaluate score     │
│  → build feedback     │
│  (repeat ≤ N iters)   │
└───────┬───────────────┘
        │
   calls│
   ┌────▼──────────┐   ┌────────────────────┐   ┌───────────────┐
   │ arc_agi/llm.py│   │arc_agi/sandbox.py  │   │arc_agi/       │
   │ (LLM calls    │   │(run generated code │   │scoring.py     │
   │  via LiteLLM, │   │ in a subprocess    │   │(exact-match   │
   │  rate-limited)│   │ with a timeout)    │   │ scoring)      │
   └───────────────┘   └────────────────────┘   └───────────────┘
        │
  uses  │
   ┌────▼──────────┐   ┌────────────────────┐   ┌───────────────┐
   │ arc_agi/      │   │arc_agi/prompts.py  │   │arc_agi/       │
   │ types.py      │   │(system prompts and │   │config.py      │
   │ (TypedDicts)  │   │ feedback template) │   │(ExpertConfig  │
   └───────────────┘   └────────────────────┘   │ list)         │
                                                 └───────────────┘
```

---

## 2. Layers of the Architecture

The system has **four conceptual layers**, each building on the one below it:

| Layer | Files | Responsibility |
|---|---|---|
| **Entry / Orchestration** | `main.py` | Load data, drive async loop, write output |
| **Expert Coordination** | `solve.py`, `solve_parallel_coding.py` | Launch N experts, vote on results |
| **Single Expert** | `solve_coding.py` | One iterative LLM+sandbox solving loop |
| **Utilities** | `llm.py`, `sandbox.py`, `scoring.py`, `io.py`, `prompts.py`, `utils.py`, `types.py`, `config.py` | LLM calls, code execution, I/O, types |

---

## 3. Key Design Decisions (and Why They Were Made)

### 3.1 Code Generation instead of Direct Prediction

Most ARC solvers try to directly predict the output pixel grid. Poetiq instead asks the LLM to write a **Python `transform` function**. Why?

- **Generalization**: Code can capture the *rule* (e.g. "rotate 90°") rather than just the example outputs. This means the same code works on all test inputs, not just the one it was shown.
- **Verifiability**: You can run the code against training examples to check if it is correct *before* submitting. Direct pixel predictions can't be verified this way.
- **Expressiveness**: Any logical transformation can be expressed as code; pixel-by-pixel generation is fundamentally limited.

### 3.2 Iterative Refinement with Feedback

If the first code attempt is wrong, the system doesn't give up. It:
1. Runs the code on all training inputs.
2. Formats a detailed diff (cell-by-cell comparison, shape mismatch, error messages).
3. Includes this feedback in the *next* prompt, asking the LLM to improve.

This is analogous to a human programmer running tests, reading the error output, and fixing the bug. Up to `max_iterations` attempts are made per expert.

### 3.3 Multi-Expert Voting

Instead of trusting a single LLM call, the system runs **N independent experts** in parallel. Each expert independently finds its best solution. Then:
- Results are grouped by the test output they produce.
- The group with the most members wins (majority vote).
- This reduces the chance of a one-off mistake being submitted.

The number of experts is controlled by `NUM_EXPERTS` in `config.py`.

### 3.4 Sandboxed Code Execution

LLM-generated code is **never run in the main process**. It is written to a temporary file and executed in a child subprocess with:
- No environment variables (except `PYTHONHASHSEED`).
- A hard CPU timeout (default 5 seconds).
- Output captured and parsed as JSON.

This protects the solver from infinite loops, file-system access, or malicious payloads.

### 3.5 Async Throughout

The entire system uses Python `asyncio`. Every LLM call, every sandbox execution, and every task is an async coroutine. This means:
- Multiple experts for one problem run truly in parallel.
- Multiple problems can run concurrently (all tasks are created at startup with `asyncio.create_task`).
- The event loop efficiently uses I/O wait time (LLM response latency) to run other work.

### 3.6 LiteLLM for Multi-Provider Support

All LLM calls go through the `litellm` library. This means swapping from Gemini to GPT to Claude requires changing only the `llm_id` string in `config.py`. The calling code in `llm.py` never changes.

---

## 4. Module Dependency Graph

```
main.py
  ├── arc_agi/config.py
  ├── arc_agi/io.py
  ├── arc_agi/scoring.py
  └── arc_agi/solve.py
        └── arc_agi/solve_parallel_coding.py
              └── arc_agi/solve_coding.py
                    ├── arc_agi/llm.py
                    │     └── (litellm, asynciolimiter)
                    ├── arc_agi/sandbox.py
                    │     └── (asyncio subprocess)
                    ├── arc_agi/prompts.py
                    └── arc_agi/types.py
```

---

## 5. Data Structures at a Glance

```
Grid            = list[list[int]]            (2D colour grid, integers 0–9)
ExpertConfig    = TypedDict (see types.py)   (all settings for one expert)
ARCAGISolution  = TypedDict                  (one code attempt + feedback + score)
ARCAGIResult    = TypedDict                  (final result from one expert)
RunResult       = TypedDict                  (result of running code on one example)
Message         = TypedDict {role, content}  (LLM conversation message)
```

See [modules.md](modules.md) for the full field-by-field breakdown of each TypedDict.

---

## 6. External Dependencies

| Package | Version pinned in requirements.txt | Purpose |
|---|---|---|
| `litellm` | 1.78.2 | Unified LLM API (Gemini, OpenAI, Anthropic, Grok, Groq) |
| `asynciolimiter` | 1.2.0 | Per-model rate limiting |
| `numpy` | 2.3.4 | Grid arithmetic and soft-score computation |
| `scipy` | 1.16.3 | Available inside the sandbox for generated code |
| `python-dotenv` | 1.1.1 | Load API keys from `.env` file |

No web framework, no database, no message queue. The system is a pure Python async script.
