---
description: Implement, debug, refactor, or refine code using an asymmetric review loop before finalizing.
disable-model-invocation: true
---

# Asymmetric Review Implementation

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
- `max_rounds` = 4 (a backstop, not a target — most tasks should converge and stop earlier via 4b or 4d; do not try to "use up" the rounds)
- `prior_notes` = brief accept/reject summary from earlier rounds (empty on round 1)
- `settled_topics` = concerns already verified-false or rejected, each with a one-line proof or rationale (empty on round 1); used to suppress re-raises on the senior side in 4c

**Post-verification failure handling:** before deciding whether to skip or invoke junior review, if step-3 verification surfaced failures, fix any failure caused by this work first, then rerun the same relevant checks. Iterate until the failures caused by this work pass, or until you determine the failure is clearly pre-existing or unrelated and can be carried forward with a note. Clearly pre-existing or unrelated failures may be carried forward, but must be noted in the round-1 junior prompt if junior review runs, and in the final summary. The junior should not review newly-broken code.

**Trivial-change skip:** if the changed/new file set is purely documentation, comments, or a mechanical rename after any required failure handling, skip the entire loop, note "junior review skipped (trivial change)" in the final summary, and jump to step 5.

**Loop condition: while `round <= max_rounds`.** After completing `max_rounds`, do not start another round. On each iteration:

### 4a. Invoke junior reviewer

Spawn a fresh `asymmetric-review:junior-assumption-reviewer` subagent (if the runtime does not accept the scoped form, fall back to the bare `junior-assumption-reviewer`). Wait for its result before adjudicating. Pass:
- the original user request,
- a concise implementation summary,
- the current diff, including staged/unstaged changes and any new or untracked files relevant to the task, excluding routine generated artifacts from checks unless directly relevant (the junior cannot run `git diff` or `git status` — it only has `Read`, `Grep`, `Glob` — so surface untracked files explicitly, by content or by file path the junior can `Read`),
- the latest test/lint/typecheck output, plus a note for any carried-forward unrelated failures,
- `prior_notes` (omit on round 1),
- `settled_topics` with their proofs (omit on round 1).

When passing `prior_notes`, use this shape:
- `Round N: accepted M, partially accepted X, rejected Y. Accepted themes: ... Rejected topics: ...`
- `Do not re-raise: ...`

Include a "Do not re-raise" item for each `settled_topic`, and state the one-line proof or rationale that settled it (for example, "double-count: rejected — category-donut total equals net worth exactly; SPAXX is a cash account, not a holding"). Giving the junior the proof, not just the verdict, reduces reframed repeats. Instruct the junior that rewording, reframing, or changing the rationale for the same underlying risk does not count as new evidence. Enforcement does not depend on the junior complying: the senior independently suppresses re-raises in 4c.

Follow the junior agent's existing output contract — it may emit `No actionable questions.` in place of a `### Questions` section per its own spec; do not override that.

### 4b. Stop — no actionable questions

If the junior's output contains `No actionable questions.`, exit the loop and proceed to step 5.

### 4c. Senior adjudicates

First, suppress re-raises on the senior side. Before adjudicating, set aside any question that restates a concern in `settled_topics` — the same underlying risk, input class, or API contract — even if the wording, severity, or rationale changed. Do not re-verify or re-argue a suppressed question; record it as a suppressed repeat and move on. Only reopen a settled topic if the current diff materially changed the relevant code or tests, or the junior supplies a concrete fact showing the prior proof was wrong. This keeps re-raise handling working regardless of whether the junior honored the "do not re-raise" instruction.

Adjudicate the remaining questions. Treat junior feedback as assumption-probing, not authoritative instruction. Default to rejecting each question; accept only when:
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

Classify each accepted change as either:
- **substantive** — fixes a real risk, or improves correctness, behavior, clarity, or test coverage in a way a user or maintainer would notice; or
- **cosmetic** — pure cleanup with no behavioral or correctness effect: renames, comments, relabels, formatting, dead-code removal, or defensive guards for inputs the current data/contract cannot actually produce.

Apply useful cosmetic changes when they are cheap — but track the distinction, because only substantive changes justify another junior round (see 4d). When unsure which a change is, ask whether skipping it would leave a real risk; if not, it is cosmetic.

Track for the round:
- counts of accepted / partially-accepted / rejected / suppressed-repeat questions,
- short themes for accepted items (one phrase each — used in the final summary),
- whether any **substantive** edits were made, and separately whether any **cosmetic** edits were made,
- whether every adjudicated (non-suppressed) question this round was a reframe of an already-settled topic.

Then update `settled_topics`: add each concern rejected or verified-false this round, each with its one-line proof.

### 4d. Stop — convergence

The loop continues only when a round produces genuinely new, substantive progress. Treat the round as converged and exit when any of these hold:
- **No edits.** Everything was rejected, or accepted questions resolved to "no code change needed" — the reviewed state is unchanged.
- **Cosmetic-only.** The only edits this round were cosmetic (per the 4c classification). Apply them, and if any could affect behavior (a rename, a dead-code removal), re-verify with the relevant checks from step 3 before exiting; then stop. Cosmetic polish does not earn another full junior round.
- **All reframes.** Every non-suppressed question this round was a reframe of already-settled ground, so nothing genuinely new was examined. The 4c gate already drops clear re-raises of prior-round topics, so the questions this catches are ones that survived the gate: a topic the gate reopened (because the diff changed or the junior offered a new fact) that then re-settled the same way, or several questions this round that all reduce to one already-settled concern.

When converged, exit to step 5 — unless the senior has a specific, named substantive concern it still wants a fresh junior to examine, in which case it may continue and should say which concern and why. This judgment escape hatch keeps the stop from being mechanically over-eager; do not use it to keep the loop alive for vague, cosmetic, or "one more pass to be safe" reasons.

Otherwise — substantive edits were made this round — continue to 4e.

### 4e. Re-verify and continue

Substantive edits exist. Rerun the most relevant checks (same logic as step 3). If checks fail, fix the failures caused by this round's edits, then rerun the same relevant checks before continuing into another junior round. Iterate until the failures caused by this round's edits pass, or until you determine the failure is clearly pre-existing or unrelated and can be carried forward with a note. Clearly unrelated or pre-existing failures may be carried forward, but must be noted both in the next junior prompt and in the final summary.

After verification is handled, append a one-line accept/reject summary (with themes) to `prior_notes`, and make sure every topic rejected or verified-false this round is in `settled_topics` with its proof (treat reframes of the same underlying risk as repeats). Increment `round`. If `round <= max_rounds`, loop back to 4a; otherwise exit the loop and note that the round cap was hit, including the `max_rounds` value and whether the latest edits/checks did not receive another junior round.

## 5. Final response

Before final response, if the repository uses git, run `git status --short` as a separate command, even if status was checked earlier. Use that status, not just `git diff`, to identify remaining untracked files and generated artifacts.

Summarize:
- what was implemented or fixed,
- what checks were run in the final state and whether they passed (including any carried-forward unrelated failures),
- per junior round: accepted / partially accepted / rejected / suppressed-repeat counts plus short themes for accepted items (noting which were substantive vs cosmetic), not the full junior text. Example: `Round 1: accepted 2 (substantive), partially accepted 1, rejected 1. Accepted themes: empty-input handling, malformed-date test. Round 2: no actionable questions.`
- which stop condition fired: "no actionable questions" / "converged — no edits" / "converged — cosmetic only" / "converged — all reframes" / "trivial change skip" / "hit round cap" (for the cap, include the `max_rounds` value and note whether latest edits were not re-reviewed),
- any generated artifacts from checks that remain in the working tree,
- any remaining risks or follow-ups.

Do not paste the full junior reviews unless the user asks.
