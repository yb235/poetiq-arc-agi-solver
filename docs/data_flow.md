# Data Flow

This document traces exactly what happens to every piece of data as it moves through the system — from raw JSON files on disk to the final Kaggle submission. Think of it as following a parcel through a sorting warehouse.

---

## Overview Diagram

```
  ┌────────────────────────────────────────────────┐
  │              DISK (input)                      │
  │                                                │
  │  data/arc-prize-2024/                          │
  │    arc-agi_evaluation_challenges.json          │
  │    arc-agi_evaluation_solutions.json  (opt.)   │
  └──────────────────────┬─────────────────────────┘
                         │  json.load()
                         ▼
  ┌────────────────────────────────────────────────┐
  │         In-memory Python dict                  │
  │  {task_id: {"train": [...], "test": [...]}}    │
  └──────────────────────┬─────────────────────────┘
                         │  unpack train/test
                         ▼
  ┌────────────────────────────────────────────────┐
  │  Typed Python lists (still integers)           │
  │  train_in  = list[list[list[int]]]             │
  │  train_out = list[list[list[int]]]             │
  │  test_in   = list[list[list[int]]]             │
  └──────────────────────┬─────────────────────────┘
                         │  format_problem()
                         ▼
  ┌────────────────────────────────────────────────┐
  │  ASCII-art problem string (plain text)         │
  │                                                │
  │  Example #1                                    │
  │  Input:                                        │
  │  <Diagram>                                     │
  │  0 1 2                                         │
  │  3 4 5                                         │
  │  </Diagram>                                    │
  │  ...                                           │
  └──────────────────────┬─────────────────────────┘
                         │  inserted into solver_prompt
                         ▼
  ┌────────────────────────────────────────────────┐
  │  Full LLM prompt (plain text, ~thousands of    │
  │  tokens: system instructions + examples +      │
  │  problem + optional feedback from prev tries)  │
  └──────────────────────┬─────────────────────────┘
                         │  LiteLLM acompletion()
                         ▼
  ┌────────────────────────────────────────────────┐
  │  LLM response text (plain text)                │
  │  Contains an explanation paragraph and a       │
  │  ```python ... ``` code block                  │
  └──────────────────────┬─────────────────────────┘
                         │  re.search(r"```python…```")
                         ▼
  ┌────────────────────────────────────────────────┐
  │  Python source code string                     │
  │  def transform(grid: np.ndarray) -> np.ndarray:│
  │      ...                                       │
  └──────────────────────┬─────────────────────────┘
                         │  sandbox.run() per training example
                         ▼
  ┌────────────────────────────────────────────────┐
  │  Subprocess execution                          │
  │  stdin  → JSON: {"input": [[0,1],[2,3]]}       │
  │  stdout ← JSON: {"ok": true, "result": [...]}  │
  │  stderr ← error text (if crash)                │
  └──────────────────────┬─────────────────────────┘
                         │  json.loads(stdout)
                         ▼
  ┌────────────────────────────────────────────────┐
  │  RunResult (TypedDict)                         │
  │  {success, output (JSON str), soft_score,      │
  │   error, code}                                 │
  └──────────────────────┬─────────────────────────┘
                         │  aggregated across experts
                         ▼
  ┌────────────────────────────────────────────────┐
  │  ARCAGIResult (TypedDict) per expert           │
  │  {train_results, results (test), iteration,    │
  │   prompt_tokens, completion_tokens}            │
  └──────────────────────┬─────────────────────────┘
                         │  voting + ranking
                         ▼
  ┌────────────────────────────────────────────────┐
  │  Ordered list[ARCAGIResult]                    │
  │  (best first after voting)                     │
  └──────────────────────┬─────────────────────────┘
                         │  build_kaggle_two_attempts()
                         ▼
  ┌────────────────────────────────────────────────┐
  │  Kaggle predictions (Python list)              │
  │  [{"attempt_1": grid, "attempt_2": grid}, …]  │
  └──────────────────────┬─────────────────────────┘
                         │  json.dump()
                         ▼
  ┌────────────────────────────────────────────────┐
  │              DISK (output)                     │
  │  output/submission_<TIMESTAMP>.json            │
  └────────────────────────────────────────────────┘
```

---

## 1. Input Data Format

### Challenge file (`arc-agi_evaluation_challenges.json`)

```json
{
  "b7999b51": {
    "train": [
      { "input":  [[0,1,0],[1,0,1],[0,1,0]],
        "output": [[1,0,1],[0,1,0],[1,0,1]] },
      { "input":  [[1,1,0],[0,1,0],[0,1,1]],
        "output": [[0,0,1],[1,0,1],[1,0,0]] }
    ],
    "test": [
      { "input": [[0,0,1],[1,0,0],[0,1,0]] }
    ]
  },
  ...
}
```

- Keys are task IDs (8-character hex strings).
- `train` has 2–10 input/output pairs that reveal the transformation rule.
- `test` has 1+ inputs whose outputs must be predicted.

### Solutions file (`arc-agi_evaluation_solutions.json`)

```json
{
  "b7999b51": [
    [[1,1,0],[0,1,1],[1,0,1]]
  ],
  ...
}
```

One list per task; each element is the expected output for the corresponding test input. Used only for local scoring; the hidden test set has no solutions file.

---

## 2. Grid Representation

Grids flow through the system in different forms depending on what is needed:

| Stage | Representation | Reason |
|---|---|---|
| JSON file | `list[list[int]]` (nested Python lists) | JSON only supports lists, not numpy |
| Inside LLM prompt | ASCII art text (space-separated digits) | LLMs read text, not arrays |
| Sandbox input | JSON string `{"input": [[…]]}` over stdin | Cross-process communication |
| Sandbox output | JSON string `{"ok": true, "result": [[…]]}` on stdout | Cross-process communication |
| Scoring / comparison | `numpy.ndarray` of ints | Efficient element-wise comparison |
| Final submission | `list[list[int]]` (nested Python lists) | Kaggle expects JSON-serialisable lists |

The `_coerce_grid()` function in `io.py` handles the common case where a grid might arrive as a numpy array, a JSON string, or a plain Python list, and normalises it to a plain Python list.

---

## 3. Prompt Construction — Detailed Data Flow

### 3.1 The Problem String

`format_problem(example, shuffle, seed)` in `solve_coding.py` takes the dict `{"train": [...], "test": [...]}` and renders it as human-readable text:

```
Example #1
Input:
<Diagram>
0 1 2
3 4 5
6 7 8
</Diagram>

Output:
<Diagram>
8 7 6
5 4 3
2 1 0
</Diagram>

Challenge #1
Input:
<Diagram>
1 2 3
4 5 6
7 8 9
</Diagram>
```

If `shuffle=True`, the training examples are permuted using the given seed before formatting. This makes each iteration show examples in a different order, encouraging the LLM to notice different patterns.

### 3.2 The System Prompt

`_build_prompt(solver_prompt, problem=problem_str)` replaces the `$$problem$$` placeholder in the chosen solver prompt template (from `prompts.py`) with the formatted problem string.

### 3.3 The Feedback Block (on iterations 2+)

If previous failed solutions exist, a random subset is selected (controlled by `selection_probability`). They are rendered by `create_examples()` into an XML-like block:

```xml
<solution_1>
<solution_code>
```python
def transform(grid):
    return grid[::-1]
```
</solution_code>
<solution_evaluation>
Solves Example #1 incorrectly.

Shape mismatch: your prediction's shape was (3, 3),
while the correct shape was (2, 3).
</solution_evaluation>
<solution_score>0.00</solution_score>
</solution_1>
```

This block is appended to the main prompt after the `feedback_prompt` template (which also uses a `$$feedback$$` placeholder).

### 3.4 The Final Message Sent to the LLM

```
[system_prompt_with_examples_and_problem]
[optional: feedback_prompt_with_previous_attempts]
```

The entire message is sent as a single user turn (the `messages` list has one element with `role="user"`).

---

## 4. Sandbox Data Flow — Detailed

```
solve_coding.py               u.py (subprocess)
     │                               │
     │  write to temp file           │
     ├──────────────────────────────►│
     │                               │  (LLM-generated code)
     │  stdin ← json.dumps({"input": grid})
     ├──────────────────────────────►│
     │                               │  data = json.load(stdin)
     │                               │  res = transform(np.array(data['input']))
     │                               │  print(json.dumps({"ok": True, "result": res.tolist()}))
     │  stdout → json string         │
     │◄──────────────────────────────┤
     │                               │
     │  json.loads(stdout)           │
     │  → {"ok": True, "result": [[…]]}
     │                               │
     │  _json_to_ndarray(result_str) │
     │  → numpy.ndarray              │
     │                               │
     │  _soft_score(pred, truth)     │
     │  → float 0.0–1.0              │
```

The sandbox wraps the LLM code in a minimal harness (`_build_script`) that:
1. Imports `json`, `numpy`, and `scipy` (so the generated code can use them).
2. Reads the input grid from stdin.
3. Calls `transform()`.
4. Prints the result as JSON.

---

## 5. Voting Data Flow — Detailed

After all N experts return their `ARCAGIResult`:

```
results = [result_0, result_1, ..., result_N-1]
```

For each result:

```python
is_passer = all(rr["success"] for rr in result["train_results"])
key = canonical_test_key(result["results"])
```

`canonical_test_key` (in `utils.py`) serialises the test outputs to a stable string — two results have the same key if and only if they produced identical test output grids.

```
candidate_buckets = {
    "[[1,2],[3,4]]|[[5,6],[7,8]]": [result_2, result_5, result_7],  ← 3 votes
    "[[0,0],[0,0]]|[[1,1],[1,1]]": [result_1],                      ← 1 vote
}
failure_buckets = {
    "[[9,9],[9,9]]|[[8,8],[8,8]]": [result_0, result_3],            ← 2 votes
    ...
}
```

With `count_failed_matches=True`, if a failure key matches a candidate key, the failures are merged into the candidates (boosting their vote count).

The final ordered list prioritises:
1. Most-voted candidate (diversity-first: one representative per group).
2. Most-voted failure group representatives.
3. Remaining candidate members.
4. Remaining failure members.

---

## 6. Output Data Flow

### `submission_<TIMESTAMP>.json`

```json
{
  "b7999b51": [
    {"attempt_1": [[1,1,0],[0,1,1],[1,0,1]], "attempt_2": []},
    ...
  ],
  "a1b2c3d4": [
    {"attempt_1": [[0,0,0],[0,0,0]], "attempt_2": [[1,0,0],[0,0,1]]},
    ...
  ]
}
```

This file can be uploaded directly to Kaggle. It is rewritten after every completed task so partial results are always available.

### `tokens_<TIMESTAMP>.json`

```json
{
  "b7999b51": {"prompt": 4120, "completion": 832, "total": 4952},
  "a1b2c3d4": {"prompt": 3980, "completion": 760, "total": 4740}
}
```

Used to track API costs. Token costs vary by model and provider.

### `config_<TIMESTAMP>.json`

A full snapshot of `CONFIG_LIST` for the run. Allows exact reproduction of a past run.

---

## 7. Feedback Loop Data Flow (Iterative Refinement)

The feedback loop is what turns a single LLM call into an iterative solver:

```
Iteration 1:
  solutions = []
  → prompt (no feedback)
  → LLM generates code_1
  → sandbox runs code_1 on train_0, train_1, train_2
  → results: [FAIL(soft=0.33), FAIL(soft=0.0), FAIL(soft=0.5)]
  → _build_feedback() generates feedback text
  → solutions = [ARCAGISolution(code=code_1, feedback="...", score=0.28)]

Iteration 2:
  selected = solutions (maybe subset, based on selection_probability)
  → prompt + feedback from solutions
  → LLM generates code_2 (improved)
  → sandbox runs code_2 on train_0, train_1, train_2
  → results: [PASS(soft=1.0), PASS(soft=1.0), PASS(soft=1.0)]
  → ALL PASS! → also run on test inputs → return ARCAGIResult immediately ✓
```

The feedback text is the key: it shows the LLM exactly what went wrong (cell-by-cell diff, shape mismatch, exception traceback) so the LLM can fix it on the next try.
