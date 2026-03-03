# Configuration Guide

This document explains every configurable parameter in the Poetiq ARC-AGI Solver. Each option is explained in plain English with guidance on when and why you might change it.

---

## Where Configuration Lives

All configuration is in **`arc_agi/config.py`**. Two things are defined there:

1. `NUM_EXPERTS` — how many independent experts to run in parallel.
2. `CONFIG_LIST` — a list of `ExpertConfig` dicts, one per expert.

```python
# config.py
NUM_EXPERTS = 1    # Change to 2 or 8 for higher-accuracy runs

CONFIG_LIST: list[ExpertConfig] = [
    { ... }        # all settings
] * NUM_EXPERTS
```

All entries in `CONFIG_LIST` are copies of the same dict (at least by default). You could give each expert a different LLM model or prompt if you wanted.

Additionally, **`main.py`** contains a few top-level constants that control which problems to run:

| Constant | Default | Meaning |
|---|---|---|
| `DATA_CHALLENGES` | `data/arc-prize-2024/arc-agi_evaluation_challenges.json` | Path to the puzzle file |
| `DATA_SOLUTIONS` | `data/arc-prize-2024/arc-agi_evaluation_solutions.json` | Path to the solutions file (optional, for local scoring) |
| `OUTPUT_DIR` | `output/` | Where to write submission and token files |
| `NUM_PROBLEMS` | `None` | Run only the first N problems. `None` = all. |
| `SELECTED_PROBLEMS` | `[]` | Run only these task IDs. Empty list = all. |

---

## Expert Configuration Fields

### LLM Settings

---

#### `llm_id`
**Type**: `Models` (a string)  
**Default**: `"gemini/gemini-3-pro-preview"`

Which AI model to use. The string must be one of the values defined in `types.py`:

| Value | Provider | Notes |
|---|---|---|
| `"gemini/gemini-3-pro-preview"` | Google Gemini | Current default; best performance |
| `"gemini/gemini-2.5-pro"` | Google Gemini | Previous generation; needs thinking enabled |
| `"openai/gpt-5"` | OpenAI | Reasoning effort set to "high" automatically |
| `"openai/gpt-5.1"` | OpenAI | Same as gpt-5 |
| `"anthropic/claude-sonnet-4-5"` | Anthropic | Thinking enabled (32k budget) |
| `"anthropic/claude-haiku-4-5"` | Anthropic | Faster/cheaper Claude; thinking enabled |
| `"xai/grok-4"` | xAI | Full-size Grok |
| `"xai/grok-4-fast"` | xAI | Faster/cheaper Grok |
| `"groq/openai/gpt-oss-120b"` | Groq | Fast inference for OSS model |

**How to change**: Edit the `llm_id` value in `config.py`. Make sure you have the corresponding API key in your `.env` file.

---

#### `solver_temperature`
**Type**: `float`  
**Default**: `1.0`  
**Range**: `0.0` – `2.0`

Controls how "creative" (random) the LLM's responses are.

- **0.0**: Almost deterministic. The model always picks the most likely next token. Good for reliable, reproducible behaviour.
- **1.0**: Balanced. The model explores alternatives, which is good for finding diverse code approaches.
- **2.0**: Very random. Usually produces incoherent output.

**Recommendation**: Keep at `1.0`. Lower values may cause the model to repeat the same wrong code across iterations instead of trying new approaches.

---

#### `request_timeout`
**Type**: `Optional[int]` (seconds)  
**Default**: `3600` (1 hour)

How long to wait for a single LLM API response before timing out. If the model takes longer than this, the request is cancelled.

- Very long timeouts are needed for reasoning models that think for a long time.
- For faster models, you could lower this to `300` (5 minutes) to fail faster.

---

#### `max_total_timeouts`
**Type**: `Optional[int]`  
**Default**: `15`

How many timeout errors an expert is allowed to accumulate before giving up on the problem entirely. A timeout counts only if the LLM call times out (not if the sandbox times out).

- Set to `None` to disable this limit (the expert will retry indefinitely on timeouts).
- Lower this value to make the solver fail faster when the API is slow.

---

#### `max_total_time`
**Type**: `Optional[int]` (seconds)  
**Default**: `None` (no limit)

Total time budget per expert per problem. If the sum of all LLM call durations exceeds this, the expert exits early.

- `None` means no time limit.
- Useful for budget-constrained runs where you want each problem to finish quickly.

---

#### `per_iteration_retries`
**Type**: `int`  
**Default**: `2`

How many times to retry a single LLM call if it fails with an unexpected error (not a rate-limit or timeout error).

---

### Solver Loop Settings

---

#### `max_iterations`
**Type**: `int`  
**Default**: `10`

The maximum number of times each expert will try to solve a problem before giving up. Each iteration involves one LLM call and one sandbox execution.

- More iterations = more chances to find the right answer, but also more API cost.
- If the correct answer is found before `max_iterations`, the loop exits early.

---

#### `max_solutions`
**Type**: `int`  
**Default**: `5`

Maximum number of previous failed solutions to include in the feedback prompt.

- Higher = the LLM has more context about what has already been tried.
- Too high and the prompt may become very long (expensive and slower to process).

---

#### `selection_probability`
**Type**: `float`  
**Default**: `1.0`  
**Range**: `0.0` – `1.0`

Each previous solution is independently included in the feedback prompt with this probability.

- `1.0`: Always include all selected solutions (up to `max_solutions`).
- `0.5`: Each solution has a 50% chance of being included.
- Lower values introduce randomness in the feedback, which can help with diversity.

---

#### `seed`
**Type**: `int`  
**Default**: `0`

The random seed for the numpy RNG that controls example shuffling and solution selection.

- Different seeds produce different orderings of training examples.
- When running multiple experts, `solve_parallel_coding.py` automatically offsets each expert's seed by `expert_index * max_iterations`, so all experts explore different orderings.

---

#### `shuffle_examples`
**Type**: `bool`  
**Default**: `True`

Whether to randomly shuffle the order of training examples in the prompt each iteration.

- `True`: Each iteration shows examples in a different order. This helps the LLM not fixate on the first example and discover patterns from a different perspective.
- `False`: Examples always appear in the same order.

---

#### `improving_order`
**Type**: `bool`  
**Default**: `True`

How to order previous solutions in the feedback prompt.

- `True`: Show solutions from worst to best score (ascending). The LLM sees the progression and ends with the best attempt, which can help it build on the best solution seen so far.
- `False`: Show solutions from best to worst score (descending).

---

#### `return_best_result`
**Type**: `bool`  
**Default**: `True`

What to return if no perfect solution was found after `max_iterations` iterations.

- `True`: Return the iteration that had the highest mean soft score on training examples (the "best partial result").
- `False`: Return the last iteration's result.

**Recommendation**: Keep `True`. The best partial result is almost always more useful than the last result.

---

### Voting Settings

These settings control how results from multiple experts are aggregated. They are read from `expert_configs[0]` (the first expert), so they should be the same for all experts in a run.

---

#### `use_new_voting`
**Type**: `bool`  
**Default**: `True`

Which voting algorithm to use.

- `True` (new): Diversity-first voting. Groups results by identical test output. Picks one representative per group (most-voted group first), then fills in with failure groups and remaining members.
- `False` (old): Simpler approach. Passers ordered by vote count; failures sorted flat by soft score.

**Recommendation**: Keep `True`. The new algorithm is better.

---

#### `count_failed_matches`
**Type**: `bool`  
**Default**: `True`

Whether to count failed experts (code that didn't pass training) as votes if their test output matches a passing candidate's test output.

- `True`: A failure that happens to produce the same test output as a passer gets folded into that group and boosts its vote count. This rewards consensus even from imperfect solutions.
- `False`: Failures are always kept in separate failure buckets.

---

#### `iters_tiebreak`
**Type**: `bool`  
**Default**: `False`

Whether to use the number of iterations taken as a tiebreaker when two voting groups have the same size.

- `True`: When two groups are tied on votes, the group whose representative used fewer (or more, see `low_to_high_iters`) iterations is preferred.
- `False`: No tiebreak on iterations; rely on other ordering.

---

#### `low_to_high_iters`
**Type**: `bool`  
**Default**: `False`

Only relevant when `iters_tiebreak=True`.

- `True`: Prefer solutions found in fewer iterations (found the answer faster).
- `False`: Prefer solutions found in more iterations (went through more refinement).

---

## Configuration Presets

The three Poetiq configurations mentioned in the blog post correspond to different `NUM_EXPERTS` values:

| Name | `NUM_EXPERTS` | Notes |
|---|---|---|
| Poetiq(Gemini-3-a) | `1` | Single expert. Fastest and cheapest. |
| Poetiq(Gemini-3-b) | `2` | Two experts vote. Better accuracy. |
| Poetiq(Gemini-3-c) | `8` | Eight experts vote. Best accuracy, highest cost. |

To switch between them, change the `NUM_EXPERTS` line at the top of `config.py`:

```python
# config.py

# Poetiq(Gemini-3-a):
NUM_EXPERTS = 1

# Poetiq(Gemini-3-b):
# NUM_EXPERTS = 2

# Poetiq(Gemini-3-c):
# NUM_EXPERTS = 8
```

---

## Using a Different Model

Example: switching from Gemini to GPT-5.

1. Make sure `OPENAI_API_KEY` is in your `.env`.
2. In `config.py`, change `llm_id`:

```python
CONFIG_LIST: list[ExpertConfig] = [
    {
        'solver_prompt': SOLVER_PROMPT_1,
        'feedback_prompt': FEEDBACK_PROMPT,
        'llm_id': 'openai/gpt-5',          # ← changed
        'solver_temperature': 1.0,
        ...
    },
] * NUM_EXPERTS
```

3. Note that `props` in `llm.py` automatically adds `{"reasoning_effort": "high"}` for GPT-5.

---

## Running Only a Subset of Problems

In `main.py`:

```python
# Run only the first 10 problems:
NUM_PROBLEMS = 10

# Run only specific problems:
SELECTED_PROBLEMS = ['b7999b51', 'a1b2c3d4']
```

---

## Using the 2025 Dataset

Change the data paths in `main.py`:

```python
DATA_CHALLENGES = os.path.join(os.path.dirname(__file__), "data", "arc-prize-2025", "arc-agi_evaluation_challenges.json")
DATA_SOLUTIONS  = os.path.join(os.path.dirname(__file__), "data", "arc-prize-2025", "arc-agi_evaluation_solutions.json")
```

Both 2024 and 2025 datasets are already included in the `data/` folder.
