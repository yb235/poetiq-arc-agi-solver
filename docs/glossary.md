# Glossary — Key Terms for Beginners

This glossary explains every important term used in the Poetiq ARC-AGI Solver in plain, simple English. If you read this first you will understand all the other documentation.

---

## A

### ARC-AGI
**Abstraction and Reasoning Corpus — Artificial General Intelligence**.
A benchmark made by François Chollet (creator of Keras) to test whether AI can reason in the same flexible way humans can. Each puzzle shows you a few input→output grid pairs. You must figure out the hidden rule and apply it to a new input.

### ARC Prize
An annual competition run by Kaggle that challenges teams to score as high as possible on the ARC-AGI benchmark. Prizes are awarded to top performers. The solver in this repo was used to achieve SOTA (state-of-the-art) results.

### Attempt
Kaggle's submission format allows **two guesses per test input**. The system therefore returns `attempt_1` and `attempt_2` for every test grid. A task is counted as "correct" if *either* attempt matches the ground truth.

---

## C

### Candidate Bucket
When multiple independent experts are run in parallel, their solutions are grouped by the output they produce for the test grid. A group of results that all produce the *same* test output is called a **candidate bucket**. The bucket with the most votes wins and is promoted to the top of the submission.

### Code-based Solving
The approach taken by this solver. Instead of directly predicting pixels, the LLM is asked to write a **Python `transform` function** that takes an input grid and returns the output grid. This function is then executed in a sandbox. This is more powerful than direct prediction because code can express arbitrary logic.

### Completion Tokens
The number of tokens in the text that the LLM *generated* (its reply). Used for cost tracking.

---

## E

### Expert
A single instance of the `solve_coding` solver, configured with a specific `ExpertConfig`. Multiple experts can run in parallel on the same problem. Each expert independently searches for a solution, then experts *vote* on the best answer.

### ExpertConfig
A Python `TypedDict` (a typed dictionary) that holds all settings for one expert, such as which LLM model to use, how many iterations to run, what temperature to use, etc. See [configuration.md](configuration.md) for all fields.

---

## F

### Feedback
After the LLM writes a `transform` function that is *wrong*, the system runs it on the training examples and compares the output to the expected output. The comparison is formatted as structured text (showing a diff grid, shape mismatch, errors, etc.) and given back to the LLM as context for the next attempt. This is the feedback loop that drives iterative refinement.

### Failure Bucket
A group of expert results where the generated code did **not** pass all training examples. Failure buckets are ranked separately from candidate (passing) buckets and are used as fallback attempts.

---

## G

### Grid
The fundamental data type in ARC-AGI. A **grid** is a 2D list of integers, where each integer represents a colour (0–9). Grids are rectangular but vary in size. They are stored as Python `list[list[int]]` in JSON and converted to `numpy.ndarray` for computation.

### Ground Truth (GT)
The correct answer grid for a test input. Only available for the *evaluation* and *training* datasets (not the hidden test set used by Kaggle for scoring).

---

## I

### Iteration
One round of the inner solving loop: build a prompt → call the LLM → parse the code → run it in the sandbox → evaluate. Each expert runs up to `max_iterations` iterations before giving up.

### Improving Order
When `improving_order=True`, previously tried solutions are shown to the LLM in *ascending* order of score (worst first), so the LLM can see the progression and try to improve. When `False`, solutions are shown in *descending* order (best first).

---

## K

### Kaggle Submission Format
The JSON format required by the ARC Prize competition. It is a dictionary where each key is a task ID (e.g. `"b7999b51"`) and each value is a list of `{"attempt_1": grid, "attempt_2": grid}` dicts — one per test input.

---

## L

### LiteLLM
A Python library (`litellm`) that provides a single unified API to call many different LLM providers (OpenAI, Gemini, Anthropic, xAI/Grok, Groq, etc.). This solver uses LiteLLM so it can run with multiple LLM backends without changing application code.

### LLM (Large Language Model)
An AI model (like Gemini, GPT, or Claude) that can read text and produce text. In this solver the LLM reads a description of the ARC problem and writes Python code to solve it.

### Limiter
A rate-limiter (`asynciolimiter.Limiter`) that throttles how many LLM requests can be made per second for each model. This prevents hitting API rate limits and getting errors.

---

## P

### Parallel Coding
The strategy of running multiple experts *simultaneously* (using Python `asyncio`) and then aggregating their answers. More experts → more shots at finding the right solution.

### Prompt Tokens
The number of tokens in the text *sent to* the LLM (the input). Used for cost tracking.

### Problem ID
A short hexadecimal string (e.g. `"b7999b51"`) that uniquely identifies an ARC task. Used for logging and to match submissions to ground-truth answers.

---

## R

### Rate Limit
Most LLM APIs restrict how many requests you can make per minute. The solver handles `RateLimitError` by retrying with a delay instead of crashing.

### Reasoning Effort / Thinking Budget
Some models (OpenAI's reasoning models, Anthropic's Claude, Gemini) support extended thinking or reasoning modes. The solver enables these via `props` in `llm.py` to get higher-quality responses.

### RunResult
A `TypedDict` that captures the result of executing generated code against *one* example. Fields: `success` (bool), `output` (the raw output string), `soft_score` (0–1 floating-point accuracy), `error` (error message if any), `code` (the code that was run).

---

## S

### Sandbox
A security boundary. Generated code runs in a *child subprocess* with no network access, no environment variables (except `PYTHONHASHSEED=0`), and a strict CPU timeout. This prevents runaway or malicious code from affecting the host process.

### Score / Hard Score
A binary 0 or 1 for a task: 1 if either attempt is exactly correct, 0 otherwise.

### Seed
A random-number-generator seed. Used to make the order of training examples reproducible. Different experts get different seeds so they explore different orderings of examples.

### Selection Probability
When previous failed solutions are shown to the LLM as feedback, each solution is independently included with probability `selection_probability`. This introduces diversity: not every iteration gets the full history.

### Shuffle Examples
When `shuffle_examples=True`, the training examples are shown to the LLM in a randomly shuffled order each iteration. This helps the LLM not over-fit to the first example.

### Soft Score
A floating-point number between 0.0 and 1.0 measuring how close an answer is. Computed as the fraction of *individual cells* that match the expected output. A soft score of 1.0 means the answer is exactly right. A soft score of 0.5 means half the cells are correct.

### Solution
An `ARCAGISolution` dict holding the Python code generated by the LLM for one iteration, the feedback text computed after running it, and the mean soft score it achieved on the training examples.

---

## T

### Task
One ARC-AGI puzzle, identified by a problem ID. A task has several training example pairs (input→output) and one or more test inputs for which the output must be predicted.

### Temperature
An LLM parameter (0.0–2.0) controlling how "creative" (random) the model is. Lower = more deterministic; higher = more varied. The solver typically uses 1.0 to encourage diverse solutions across iterations.

### Transform Function
The Python function the LLM must write. Its signature is always:
```python
def transform(grid: np.ndarray) -> np.ndarray:
    ...
```
The sandbox calls this function with each input grid and captures the returned output grid.

---

## V

### Voting
The mechanism by which multiple experts agree on an answer. Experts are grouped by the output they produced for the test grid. The group with the most members (votes) is ranked first. Ties are broken by soft score. This rewards consensus: if many independent experts arrived at the same answer, it is more likely to be correct.

---

## W

### Weighted Soft Score
When ranking failure buckets, the `_mean_soft` helper computes the mean soft score across all training examples for a result. This is used to rank imperfect solutions: a solution that gets most cells right is better than one that gets nothing right.
