# Module Reference

This document describes every file in the `arc_agi/` package, function by function and type by type. Start here when you want to understand a specific piece of code.

---

## Package Overview

```
arc_agi/
├── __init__.py            (empty – marks directory as a Python package)
├── config.py              (expert configuration list)
├── io.py                  (Kaggle output formatting)
├── llm.py                 (LLM API calls via LiteLLM)
├── prompts.py             (prompt templates)
├── sandbox.py             (safe subprocess code execution)
├── scoring.py             (exact-match task scoring)
├── solve.py               (top-level solve() entry point)
├── solve_coding.py        (single-expert iterative solver)
├── solve_parallel_coding.py (multi-expert parallel solver + voting)
├── types.py               (all TypedDict definitions)
└── utils.py               (small utility helpers)
```

---

## `config.py`

**Purpose**: Defines the list of expert configurations that `solve()` will use.

### Constants

| Name | Type | Description |
|---|---|---|
| `NUM_EXPERTS` | `int` | How many independent experts to run in parallel. Currently set to `1` (Poetiq 3-a config). Uncomment `2` or `8` for higher-performance variants. |
| `CONFIG_LIST` | `list[ExpertConfig]` | A list of `NUM_EXPERTS` copies of the default expert configuration. See [configuration.md](configuration.md) for every field. |

**Important**: The list is created with `[{...}] * NUM_EXPERTS`, which makes shallow copies. `solve_parallel_coding.py` calls `.copy()` before mutating each config's seed to avoid aliasing.

---

## `io.py`

**Purpose**: Converts internal `ARCAGIResult` objects into the Kaggle submission format.

### Functions

#### `_coerce_grid(x: Any) -> list`
Converts any grid-like value to a plain Python `list`. Handles:
- `numpy.ndarray` → calls `.tolist()`
- `str` that looks like JSON → parses it
- `list` → returns as-is
- Anything else → returns `[]`

This is a defensive helper to deal with the fact that grids can arrive in different types at different points in the pipeline.

#### `build_kaggle_two_attempts(results: list[ARCAGIResult], test_in: list[list[list[int]]]) -> list[dict]`

Converts the ordered list of expert results into the Kaggle two-attempt format.

**Algorithm**:
- For each test input index `j` (0, 1, 2, …):
  - Walk through `results` in order.
  - For each result, look at `result["results"][j]["output"]`.
  - Coerce to a list and collect if non-empty.
  - Stop after collecting 2 attempts.
  - Pad with `[]` if fewer than 2 were found.
- Return: `[{"attempt_1": grid_or_empty, "attempt_2": grid_or_empty}, ...]`

**Why two attempts?** Kaggle's scoring counts a task as correct if either attempt matches the ground truth. By submitting the top-2 candidates, the solver gets two shots per test input.

---

## `llm.py`

**Purpose**: Makes async LLM API calls with rate limiting, retries, and timeout budget management.

### Module-level State

| Name | Type | Description |
|---|---|---|
| `RETRIES` | `int` | Default maximum retry count per `llm()` call (3). |
| `RETRY_DELAY_SEC` | `int` | Seconds to sleep between retries (5). |
| `limiters` | `dict[Models, Limiter]` | One `asynciolimiter.Limiter` per supported model. Controls the maximum requests per second. Gemini 2.5 Pro gets 2 req/s; everything else gets 1 req/s. |
| `props` | `dict[Models, dict]` | Extra keyword arguments passed to `acompletion()` for each model. Used to enable extended thinking/reasoning modes. |

### Functions

#### `async llm(model, message, temperature, request_timeout, max_remaining_time, max_remaining_timeouts, problem_id, retries) -> tuple`

Makes one LLM request (with retries).

**Parameters**:
- `model`: LiteLLM model string (e.g. `"gemini/gemini-3-pro-preview"`).
- `message`: The full prompt text (a single string).
- `temperature`: Float 0–2, controls creativity.
- `request_timeout`: Hard timeout in seconds for a single request.
- `max_remaining_time`: If set, caps `request_timeout` so the total budget isn't exceeded.
- `max_remaining_timeouts`: If set, counts how many timeouts the problem is allowed before giving up early.
- `problem_id`: Used only for log messages.
- `retries`: Number of attempts before giving up (default `RETRIES`).

**Returns**: `(response_text, duration_s, updated_max_remaining_time, updated_max_remaining_timeouts, prompt_tokens, completion_tokens)`

**Error handling**:
- `RateLimitError`, `InternalServerError`, `ServiceUnavailableError`, `APIConnectionError`, `APIError`, `RouterRateLimitError`, `RouterRateLimitErrorBasic`: retry indefinitely (do not count against `retries`).
- Timeout errors: decrement `max_remaining_timeouts`; if budget exhausted, raise `RuntimeError("Exceeded timeouts allotted to the request")`.
- All other errors: count against `retries`. After `retries` attempts, re-raise.

---

## `prompts.py`

**Purpose**: Contains all prompt text as Python string constants.

### Constants

| Name | Description |
|---|---|
| `SOLVER_PROMPT_1` | Concise, structured prompt guiding the LLM step-by-step (5 sections: Analyse, Hypothesise, Implement, Test, Output). Includes 3 worked examples. |
| `SOLVER_PROMPT_2` | More detailed prompt with 4 parts: Initial Analysis, Iterative Testing, Coding Guidelines, Output Requirements. Includes 1 worked example and explicitly permits `cv2` (OpenCV). |
| `SOLVER_PROMPT_3` | Similar to SOLVER_PROMPT_2 but with 3 worked examples and emphasis on concise code. |
| `FEEDBACK_PROMPT` | Template appended after a solver prompt when providing feedback from previous failed attempts. Contains `$$feedback$$` placeholder. |

All prompts use `$$problem$$` and `$$feedback$$` as placeholders that are replaced by `_build_prompt()` in `solve_coding.py`.

---

## `sandbox.py`

**Purpose**: Executes LLM-generated Python code safely in an isolated subprocess.

### Functions

#### `async run(code: str, input_grid: list[list[int]], timeout_s: float = 1.5) -> tuple[bool, str]`

Runs `code` against `input_grid` and returns `(ok, output_or_error)`.

**Steps**:
1. Call `_build_script(code)` to wrap the code in a harness.
2. Write to a temp file `u.py` in a `tempfile.TemporaryDirectory()`.
3. Spawn a subprocess: `python u.py` with `stdin=PIPE, stdout=PIPE, stderr=PIPE`.
   - `cwd` is the temp dir (so the script can't import anything from the repo).
   - `env={"PYTHONHASHSEED": "0"}` (no other env vars — API keys can't leak).
4. Feed `json.dumps({"input": input_grid})` to stdin.
5. Wait up to `timeout_s` seconds for the process to finish.
6. On timeout: kill process, return `(False, "timeout")`.
7. On non-zero exit: return `(False, stderr_or_stdout)`.
8. On success: parse stdout JSON, return `(True, json.dumps(result))`.

#### `_build_script(code: str) -> str`
Wraps the raw LLM code in a minimal Python harness that reads a grid from stdin and writes the result to stdout.

---

## `scoring.py`

**Purpose**: Computes exact-match accuracy for local evaluation.

### Functions

#### `grids_equal(a, b) -> bool`
Returns `True` if two grids (nested lists) are structurally identical. Equivalent to `a == b` for Python lists.

#### `score_task(kaggle_preds: list[dict], gt_outputs: list) -> float`

Returns the fraction of test inputs that were answered correctly.

- `kaggle_preds`: the `[{"attempt_1": grid, "attempt_2": grid}, ...]` list.
- `gt_outputs`: the list of correct grids.
- A test input is correct if `attempt_1 == gt` OR `attempt_2 == gt`.
- Returns `correct / len(gt_outputs)` (0.0–1.0).

---

## `solve.py`

**Purpose**: Thin dispatch layer that provides the public `solve()` API.

### Functions

#### `async solve(train_in, train_out, test_in, problem_id) -> list[ARCAGIResult]`

**Parameters**:
- `train_in`: Training input grids.
- `train_out`: Training output grids.
- `test_in`: Test input grids.
- `problem_id`: Optional string for logging.

Reads `CONFIG_LIST` from `config.py`, makes a copy of each config (to avoid mutation), and delegates to `solve_parallel_coding()`.

---

## `solve_coding.py`

**Purpose**: The single-expert iterative LLM+sandbox solver. This is the core algorithm.

### Functions

#### `async solve_coding(*, train_in, train_out, test_in, config, problem_id) -> ARCAGIResult`

The main solving loop. See [workflow.md](workflow.md) Step 5 for the full narrative. Key logic:

- `rng = np.random.default_rng(seed)` — reproducible randomness for solution selection.
- `solutions: list[ARCAGISolution]` — accumulates all code attempts and their feedback.
- On each iteration:
  - `_make_example()` + `format_problem()` build the problem text.
  - `_build_prompt()` inserts the problem into the solver template.
  - `create_examples()` renders previous solutions as feedback (if any).
  - `await llm(...)` calls the LLM.
  - `_parse_code_from_llm()` extracts the Python code block.
  - `await _eval_on_train_and_test()` runs the code in the sandbox.
  - If all training examples pass → return success immediately.
  - Otherwise → build feedback, store solution, continue.
- After the loop: return best partial result (if `return_best_result=True`).

#### `create_examples(solutions, max_examples, improving_order) -> str`
Renders up to `max_examples` solutions (selected by best score) as XML-like blocks for inclusion in the feedback prompt.

#### `_build_prompt(base_prompt: str, **fields: str) -> str`
Replaces `$$key$$` placeholders in `base_prompt` with the corresponding keyword argument values.

#### `_array_diff(arr1, arr2) -> str`
Generates a human-readable diff of two 2D numpy arrays. Matching cells show the value as-is; mismatched cells show `predicted/correct`.

#### `_parse_code_from_llm(response: str) -> Optional[str]`
Extracts the content of the first ` ```python ... ``` ` block from an LLM response using a regex.

#### `_soft_score(pred, truth) -> float`
Element-wise accuracy: fraction of cells where `pred == truth`. Returns 0.0 if shapes differ.

#### `_json_to_ndarray(s: str) -> Optional[np.ndarray]`
Parses a JSON string to a numpy int array. Ensures at least 2 dimensions (ARC grids are always 2D).

#### `_make_example(train_in, train_out, test_in) -> dict`
Combines the three separate lists back into the `{"train": [...], "test": [...]}` format.

#### `format_problem(problem, shuffle, seed) -> str`
Formats the problem dict as ASCII-art text. Optionally shuffles training examples.

#### `_example_to_diagram(example) -> str`
Converts a single grid (list of lists or numpy array) to a space-separated multi-line string.

#### `async _eval_on_train_and_test(code, train_in, train_out, test_in, *, timeout_s) -> tuple[list[RunResult], list[RunResult]]`
Runs `code` in the sandbox against every training and every test input. Returns `(train_results, test_results)`.

#### `_parse_json_array_no_expand(s: str) -> Optional[np.ndarray]`
Like `_json_to_ndarray` but does not expand dimensions — preserves the original rank for comparison in `_build_feedback`.

#### `_build_feedback(train_results, train_in, train_out) -> tuple[str, float]`
Builds human-readable feedback text and a mean score for a failed attempt. For each training example it either reports a pass, or shows the shape mismatch, diff grid, and error message.

---

## `solve_parallel_coding.py`

**Purpose**: Runs N experts in parallel and aggregates results via voting.

### Functions

#### `async solve_parallel_coding(*, train_in, train_out, test_in, expert_configs, problem_id) -> list[ARCAGIResult]`

1. Offsets each config's seed: `cfg["seed"] += it * cfg["max_iterations"]`.
2. Creates one `asyncio.Task` per config → all run concurrently.
3. `asyncio.gather()` waits for all tasks.
4. Groups results into `candidate_buckets` (passed training) and `failure_buckets` (failed training) keyed by `canonical_test_key`.
5. If `use_new_voting=True` (default):
   - Optionally merges failure-bucket members into matching candidate buckets (`count_failed_matches`).
   - Optionally sorts within groups by iteration count (`iters_tiebreak`).
   - Sorts groups by size (most votes first).
   - Interleaves: one representative per group (diversity), then failures, then remaining members.
6. Returns the ordered `list[ARCAGIResult]`.

#### `_mean_soft(res: ARCAGIResult) -> float`
Computes the mean soft score across all training results in a single `ARCAGIResult`. Used to rank failure groups.

---

## `types.py`

**Purpose**: All shared type definitions.

### `Models`
A `Literal` union of all supported LLM model strings. Used for type-checking in `llm.py`.

### `ExpertConfig` (TypedDict)

| Field | Type | Description |
|---|---|---|
| `solver_prompt` | `str` | Which solver prompt template to use |
| `feedback_prompt` | `str` | Which feedback prompt template to use |
| `llm_id` | `Models` | Which LLM to use |
| `solver_temperature` | `float` | LLM temperature |
| `request_timeout` | `Optional[int]` | Max seconds per LLM request |
| `max_total_timeouts` | `Optional[int]` | Max timeouts before early exit |
| `max_total_time` | `Optional[int]` | Max total seconds per expert |
| `per_iteration_retries` | `int` | Retry attempts per LLM call |
| `num_experts` | `int` | (stored for reference) |
| `max_iterations` | `int` | Max solving iterations |
| `max_solutions` | `int` | Max old solutions shown as feedback |
| `selection_probability` | `float` | Probability of including each old solution |
| `seed` | `int` | RNG seed for example shuffling |
| `shuffle_examples` | `bool` | Shuffle training examples each iteration |
| `improving_order` | `bool` | Show feedback in ascending score order |
| `return_best_result` | `bool` | Return best partial result on failure |
| `use_new_voting` | `bool` | Use diversity-first voting (vs. old flat voting) |
| `count_failed_matches` | `bool` | Merge failures into matching candidate buckets |
| `iters_tiebreak` | `bool` | Break voting ties by iteration count |
| `low_to_high_iters` | `bool` | When tiebreaking, prefer fewer iterations |

### `Message` (TypedDict)

| Field | Type |
|---|---|
| `role` | `Literal["user", "assistant", "system"]` |
| `content` | `str` |

### `RunResult` (TypedDict)

| Field | Type | Description |
|---|---|---|
| `success` | `bool` | Did the output exactly match the expected output? |
| `output` | `str` | The raw output JSON string from the sandbox |
| `soft_score` | `float` | Cell-wise accuracy 0.0–1.0 |
| `error` | `Optional[str]` | Error message if the code crashed or produced bad output |
| `code` | `str` | The Python code that was executed |

### `ARCAGIResult` (TypedDict)

| Field | Type | Description |
|---|---|---|
| `train_results` | `list[RunResult]` | One RunResult per training example |
| `results` | `list[RunResult]` | One RunResult per test input |
| `iteration` | `int` | Which iteration produced this result (1-based) |
| `prompt_tokens` | `Optional[int]` | Total prompt tokens for this expert |
| `completion_tokens` | `Optional[int]` | Total completion tokens for this expert |

### `ARCAGISolution` (TypedDict)

| Field | Type | Description |
|---|---|---|
| `code` | `str` | The Python code for this attempt |
| `feedback` | `str` | The feedback text generated after evaluating the code |
| `score` | `float` | The mean soft score on training examples |

---

## `utils.py`

**Purpose**: Small utility functions shared across modules.

### `canonical_test_key(results: list[RunResult]) -> str`

Produces a string key that uniquely identifies a particular set of test outputs. Two `ARCAGIResult` objects with the same test outputs will produce the same key.

Used by `solve_parallel_coding.py` to group results into voting buckets.
