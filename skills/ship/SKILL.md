---
description: Implement, debug, refactor, or refine code using an asymmetric junior-review loop before finalizing.
---

# Asymmetric Code Implementation

User request:

"$ARGUMENTS"

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

Inspect the working-tree diff after editing.

## 3. Verify

Run relevant checks when obvious from the repository:
- targeted tests,
- typecheck,
- lint,
- build,
- or the nearest lightweight equivalent.

If the correct command is not obvious, say so and skip the check rather than running an unknown command.

## 4. Invoke junior review

If the diff is purely documentation, comments, or a mechanical rename, skip this step and note "junior review skipped (trivial change)" in the final summary.

Otherwise, invoke the `junior-assumption-reviewer` subagent. Pass:
- the original user request,
- a concise implementation summary,
- the working-tree diff or changed files,
- relevant test/lint/typecheck output.

Ask it to return `## Junior Review` with `### What looks solid` and `### Questions`.

## 5. Senior adjudication

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

## 6. Final verification

If step-5 refinements modified executable code or tests, rerun the most relevant checks. Otherwise skip. Do not re-invoke `junior-assumption-reviewer`.

## 7. Final response

Summarize:
- what was implemented or fixed,
- what checks were run and whether they passed,
- which junior-review questions were accepted/rejected,
- any remaining risks or follow-ups.

Do not paste the entire junior review unless the user asks.
