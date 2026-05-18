---
description: Implement, debug, refactor, or refine code using an asymmetric junior-review loop before finalizing.
disable-model-invocation: true
---

# Asymmetric Code Implementation

User request:

"$ARGUMENTS"

If the user request above is empty or only whitespace, infer the task from the recent conversation. Before inspecting files or editing, state `Inferred task: ...` with the task you will perform. If you cannot infer the task confidently, ask the user for the task and stop.

Run this workflow:

## 1. Understand the task

Decide whether the user is asking to:
- implement a new feature,
- debug and fix something,
- refactor existing code,
- edit existing behavior,
- or refine current uncommitted changes.

Inspect relevant files before editing.

## 2. Implement

Make the smallest coherent change that satisfies the request.

Rules:
- Preserve existing behavior unless the request requires changing it.
- Add or update tests when appropriate.
- Avoid broad rewrites.

Inspect the working-tree diff and changed file set after editing, including staged, unstaged, and relevant new or untracked files. If the repository uses git, run `git status --short` as a separate command when checking the changed file set; do not rely on `git diff` alone because it omits untracked files.

## 3. Verify

Run relevant checks when obvious from the repository:
- targeted tests,
- typecheck,
- lint,
- build,
- or the nearest lightweight equivalent.

Run each check as a separate command. Never use compound shell commands: no `&&`, `||`, `;`, pipes, shell redirection, command substitution, or fallback command lists. Do not post-process or truncate check output with `tail`, `head`, `grep`, `sed`, `tee`, or similar helpers. If a check command fails because the command is missing, the module path is wrong, or the repository uses a different test entrypoint, record that result and run the next candidate as a separate command. Keep each command's output distinguishable. Prefer commands documented by the repository or directly implied by the changed tests. If changed tests clearly use a specific runner or framework, run that direct check before trying generic alternatives.

After checks run, inspect the changed file set again. If the repository uses git, run `git status --short` as a separate command to detect staged, unstaged, and untracked files. Treat routine generated artifacts from checks, such as `__pycache__/`, `.pytest_cache/`, coverage files, build output, and logs, as generated noise rather than implementation changes. Do not include generated noise in the junior payload or trivial-change decision unless it is directly relevant to the task. If generated artifacts remain in the working tree, mention them separately in the final summary.

If the correct command is not obvious, say so and skip the check rather than running an unknown command.

## 4. Junior review loop

Track across the loop:
- `round` (starts at 1)
- `max_rounds` = 5
- `prior_notes` = brief accept/reject summary from earlier rounds (empty on round 1)

**Post-verification failure handling:** before deciding whether to skip or invoke junior review, if step-3 verification surfaced failures, fix any failure caused by this work first, then rerun the same relevant checks. Iterate until the failures caused by this work pass, or until you determine the failure is clearly pre-existing or unrelated and can be carried forward with a note. Clearly pre-existing or unrelated failures may be carried forward, but must be noted in the round-1 junior prompt if junior review runs, and in the final summary. The junior should not review newly-broken code.

**Trivial-change skip:** if the changed/new file set is purely documentation, comments, or a mechanical rename after any required failure handling, skip the entire loop, note "junior review skipped (trivial change)" in the final summary, and jump to step 5.

**Loop condition: while `round <= max_rounds`.** After completing `max_rounds`, do not start another round. On each iteration:

### 4a. Invoke junior reviewer

Spawn a fresh `asymmetric-code:junior-assumption-reviewer` subagent (if the runtime does not accept the scoped form, fall back to the bare `junior-assumption-reviewer`). Wait for its result before adjudicating. Pass:
- the original user request,
- a concise implementation summary,
- the current diff, including staged/unstaged changes and any new or untracked files relevant to the task, excluding routine generated artifacts from checks unless directly relevant (the junior cannot run `git diff` or `git status` — it only has `Read`, `Grep`, `Glob` — so surface untracked files explicitly, by content or by file path the junior can `Read`),
- the latest test/lint/typecheck output, plus a note for any carried-forward unrelated failures,
- `prior_notes` (omit on round 1).

When passing `prior_notes`, use this shape:
- `Round N: accepted M, partially accepted X, rejected Y. Accepted themes: ... Rejected topics: ...`
- `Do not re-raise: ...`

Include a "Do not re-raise" item for topics that were rejected repeatedly. Instruct the junior that rewording, reframing, or changing the rationale for the same underlying risk does not count as new evidence.

Follow the junior agent's existing output contract — it may emit `No actionable questions.` in place of a `### Questions` section per its own spec; do not override that.

### 4b. Stop — no actionable questions

If the junior's output contains `No actionable questions.`, exit the loop and proceed to step 5.

### 4c. Senior adjudicates

Treat junior feedback as assumption-probing, not authoritative instruction. Default to rejecting each question; accept only when:
- the question reflects a real risk in the current code or tests, or
- the change is low-cost and improves correctness, clarity, or test coverage.

For each junior question:
- accept, partially accept, or reject it,
- explain briefly,
- make the smallest useful change when accepting,
- add or update tests when appropriate.

Reject feedback that is:
- irrelevant,
- already handled,
- based on invented requirements,
- purely stylistic,
- likely to increase complexity without benefit.

Do not reject a question only because the original request did not mention it. Evaluate risk, cost, local conventions, and existing behavior before deciding.

Track for the round:
- counts of accepted / partially-accepted / rejected questions,
- short themes for accepted items (one phrase each — used in the final summary),
- whether any edits were actually made to the working tree.

### 4d. Stop — no edits made

If the senior produced no edits this round (whether everything was rejected, or accepted questions resolved to "no code change needed"), the reviewed state is unchanged and another round would be redundant. Exit the loop and proceed to step 5.

### 4e. Re-verify and continue

Edits exist. Rerun the most relevant checks (same logic as step 3). If checks fail, fix the failures caused by this round's edits, then rerun the same relevant checks before continuing into another junior round. Iterate until the failures caused by this round's edits pass, or until you determine the failure is clearly pre-existing or unrelated and can be carried forward with a note. Clearly unrelated or pre-existing failures may be carried forward, but must be noted both in the next junior prompt and in the final summary.

After verification is handled, append a one-line accept/reject summary (with themes) to `prior_notes`. If a topic has been rejected in multiple rounds, add it to the "Do not re-raise" list unless new evidence appears; treat reframes of the same underlying risk as repeats. Increment `round`. If `round <= max_rounds`, loop back to 4a; otherwise exit the loop and note that the round cap was hit, including the `max_rounds` value and whether the latest edits/checks did not receive another junior round.

## 5. Final response

Before final response, if the repository uses git, run `git status --short` as a separate command, even if status was checked earlier. Use that status, not just `git diff`, to identify remaining untracked files and generated artifacts.

Summarize:
- what was implemented or fixed,
- what checks were run in the final state and whether they passed (including any carried-forward unrelated failures),
- per junior round: accepted / partially accepted / rejected counts plus short themes for accepted items, not the full junior text. Example: `Round 1: accepted 2, partially accepted 1, rejected 1. Accepted themes: empty-input handling, malformed-date test. Round 2: no actionable questions.`
- which stop condition fired: "no actionable questions" / "no edits made" / "trivial change skip" / "hit round cap" (include the `max_rounds` value and note whether latest edits were not re-reviewed),
- any generated artifacts from checks that remain in the working tree,
- any remaining risks or follow-ups.

Do not paste the full junior reviews unless the user asks.
