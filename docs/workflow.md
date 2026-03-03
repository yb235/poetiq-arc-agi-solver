# End-to-End Workflow

This document walks you through every step that happens when you run `python main.py`, from the moment the script starts to the moment a Kaggle-ready JSON file lands in the `output/` folder.

---

## Step 0 — Before You Run

### Prerequisites

1. **Python 3.11+** installed.
2. A virtual environment activated (`python -m venv .venv && source .venv/bin/activate`).
3. Dependencies installed (`pip install -r requirements.txt`).
4. A `.env` file in the project root with at least one API key:
   ```
   GEMINI_API_KEY=...
   OPENAI_API_KEY=...
   ```
5. Optionally edit the constants at the top of `main.py`:
   - `DATA_CHALLENGES` — which puzzle file to use (default: 2024 evaluation set).
   - `DATA_SOLUTIONS` — optional ground-truth file for local scoring.
   - `NUM_PROBLEMS` — limit to the first N problems (useful for quick tests).
   - `SELECTED_PROBLEMS` — run only specific task IDs.

---

## Step 1 — Startup (`main.py`)

```
python main.py
```

1. `load_dotenv()` reads `.env` and puts API keys into environment variables.
2. A timestamp (`TIMESTAMP`) is generated (e.g. `2025-06-01_12-00-00`). All output files use this timestamp so multiple runs never overwrite each other.
3. The OS file-handle limit is raised to 65 536 with `resource.setrlimit`. This is needed because hundreds of async subprocesses may run simultaneously.
4. The `output/` directory is created (`os.makedirs`).
5. The current `CONFIG_LIST` is saved to `output/config_<TIMESTAMP>.json` for reproducibility.
6. The challenge file (JSON) is loaded into memory as a dict: `{task_id: task_dict}`.
7. If a solutions file exists, it is loaded too (enables local accuracy scoring).
8. The list of `(task_id, task_dict)` pairs is filtered/truncated according to `SELECTED_PROBLEMS` and `NUM_PROBLEMS`.

---

## Step 2 — Task Fan-Out (async)

```python
tasks = [asyncio.create_task(_eval_task_data(task_id, task)) for task_id, task in items]
```

All problems are submitted to the event loop **at the same time** as async tasks. They do not run sequentially; the event loop switches between them whenever one is waiting for an LLM response.

---

## Step 3 — Solving One Problem (`_eval_task_data`)

For each `(task_id, task)` pair, the following steps run asynchronously:

```
task_dict
  ├── train: [{input, output}, ...]
  └── test:  [{input}, ...]
```

The training inputs, training outputs, and test inputs are extracted into separate lists and passed to `solve(train_in, train_out, test_in, problem_id)`.

---

## Step 4 — Expert Dispatch (`solve.py` → `solve_parallel_coding.py`)

`solve()` reads `CONFIG_LIST` and calls `solve_parallel_coding()` with one config per expert.

Inside `solve_parallel_coding()`:

1. Each expert config gets its own seed offset to ensure different random orderings across experts.
2. `asyncio.create_task(solve_coding(...))` is called once per expert — they all run **concurrently**.
3. `asyncio.gather(*tasks)` waits for all experts to finish.

---

## Step 5 — One Expert's Solving Loop (`solve_coding.py`)

This is the heart of the system. Each expert runs an iterative loop up to `max_iterations` times:

```
┌─────────────────────────────────────────┐
│  Iteration i (i = 0 … max_iterations-1) │
│                                         │
│  1. Build the problem string            │
│     (format training examples + test    │
│      input as ASCII art grids)          │
│                                         │
│  2. Optionally append feedback from     │
│     previous failed solutions           │
│                                         │
│  3. Call the LLM (async)                │
│     → get back text containing          │
│       a Python code block               │
│                                         │
│  4. Extract the ```python ... ```       │
│     block from the response             │
│                                         │
│  5. Run the code in the sandbox         │
│     against ALL training inputs         │
│                                         │
│  6a. If ALL training checks pass:       │
│      → also run code on test inputs     │
│      → return immediately (success!)    │
│                                         │
│  6b. Otherwise:                         │
│      → compute feedback (diff grid,     │
│        shape mismatch, error messages)  │
│      → store as a "solution" candidate  │
│      → track best result seen so far    │
│      → continue to iteration i+1        │
└─────────────────────────────────────────┘
```

After the loop, if `return_best_result=True` and no perfect solution was found, the best partial result (highest mean soft score on training) is returned.

---

## Step 6 — LLM Call (`llm.py`)

Each LLM call goes through the following pipeline:

```
await limiters[model].wait()   ← rate limiter: max N req/sec per model
      ↓
acompletion(model, messages, temperature, timeout, ...)   ← LiteLLM
      ↓
On success:  return (response_text, duration, remaining_time, remaining_timeouts, prompt_tokens, completion_tokens)
On RateLimitError / ServerError:  retry indefinitely (no deduction from attempt counter)
On Timeout:  decrement max_remaining_timeouts; if exhausted, raise early-exit exception
On other error:  retry up to `per_iteration_retries` times, then re-raise
```

The function carefully tracks remaining time and timeout budget so the expert can exit early if the problem's time budget is spent.

---

## Step 7 — Sandbox Execution (`sandbox.py`)

```python
ok, output_str = await run(code, input_grid, timeout_s=5)
```

Internally:

1. A temporary directory is created.
2. The LLM's code plus a small harness is written to `u.py`:
   ```python
   # (LLM-generated code here)
   if __name__ == '__main__':
       import json, numpy as np, scipy
       data = json.load(stdin)
       res = transform(np.array(data['input']))
       print(json.dumps({"ok": True, 'result': res.tolist()}))
   ```
3. A subprocess runs `python u.py` with the input grid fed over stdin as JSON.
4. `asyncio.wait_for` enforces the timeout. If it fires, the process is killed and `(False, "timeout")` is returned.
5. If the process exits with code 0, the JSON stdout is parsed and `(True, result_json)` is returned.
6. If the process exits non-zero, the stderr (or stdout) is returned as the error message.

---

## Step 8 — Voting and Result Aggregation (`solve_parallel_coding.py`)

After all experts finish:

```
results = [ARCAGIResult from expert 1, ARCAGIResult from expert 2, ...]
```

**Passing candidates** (code solved ALL training examples):
- Grouped into buckets by the JSON key of their test output (`canonical_test_key`).
- Sorted by bucket size (most votes first).
- `count_failed_matches=True`: failures whose test output matches a passing bucket are merged in, boosting the vote count.

**Failing candidates** (code did not solve all training examples):
- Grouped by test output.
- Within each group, sorted by mean soft score (best first).
- Groups sorted by size, then best member's score.

**Final ordering**:
```
[best_passing_group_rep, 2nd_passing_group_rep, ...]  ← diversity first
[best_failing_group_rep, ...]                         ← failure fallbacks
[remaining members of passing groups]
[remaining members of failing groups]
```

This ordered list becomes `results: list[ARCAGIResult]`.

---

## Step 9 — Building the Kaggle Submission (`io.py`)

```python
kaggle_preds = build_kaggle_two_attempts(results, test_in)
```

For each test input `j`:
1. Walk through `results` in order.
2. Collect up to 2 non-empty output grids for test input `j`.
3. If fewer than 2 are found, pad with empty lists `[]`.

Result format:
```json
[
  {"attempt_1": [[1,2],[3,4]], "attempt_2": [[1,2],[3,4]]},
  ...
]
```

---

## Step 10 — Scoring (optional, `scoring.py`)

If a solutions file was provided, the local accuracy is computed in real time:

```python
task_score = score_task(kaggle_preds, gt_outputs)
```

`score_task` returns 1.0 if `attempt_1 == ground_truth` OR `attempt_2 == ground_truth` for *every* test input in the task, otherwise a partial score. Running accuracy is printed to the console as tasks complete.

---

## Step 11 — Writing Output

After each task completes (not just at the end!), the submission dict is written to disk:

```
output/
├── submission_<TIMESTAMP>.json   ← Kaggle submission (updated after each task)
├── tokens_<TIMESTAMP>.json       ← Token usage per task (cost tracking)
└── config_<TIMESTAMP>.json       ← ExpertConfig snapshot for reproducibility
```

Writing after every task means you never lose partial results if the run crashes.

---

## Step 12 — Final Summary

When all tasks have completed, a summary is printed:

```
=== Summary ===
Data file: data/arc-prize-2024/arc-agi_evaluation_challenges.json
Problems: 400
Correct: 247
Incorrect: 153
Accuracy: 61.750
Total time: 3842s
```

---

## Timeline Diagram

```
t=0  main.py starts
      │
      ├─ load data files
      ├─ create all async tasks (all problems submitted at once)
      │
t=?  [asyncio event loop runs — problems solved concurrently]
      │
      ├─ problem A: expert 1 calls LLM ... waits ... gets code ... sandbox ... ✓ done
      ├─ problem B: expert 1 calls LLM ... waits ... fails ... LLM again ... ✓ done
      ├─ problem A result written to submission.json
      ├─ problem B result written to submission.json
      ├─ ...
      │
t=end  final write + summary printed
```

The async model means the solver spends almost no time waiting — while one expert is waiting for an LLM response, other experts and other problems are making progress.
