# Changelog

All notable changes to this plugin are documented here.
Versions follow [semantic versioning](https://semver.org); users receive
plugin updates only when the version in `plugin.json` is bumped.

## [0.2.0] - 2026-07-01

Review-loop and verification improvements.

### Changed
- Verification prefers checks that exercise the changed behavior over ones
  that only prove it compiles. A green build or lint is not treated as
  evidence the change works, and anything only checked statically is
  flagged as unverified.
- When a repository declares a canonical check command (a test, lint, or
  typecheck script, a Makefile target, or a documented command), the
  workflow runs it as written at least once instead of relying only on a
  hand-built equivalent.
- Each junior round reviews the complete change set against the original
  request, not only the most recent round's edits, so parts written first
  are not crowded out by later fixes.
- An accepted fix resolves the whole class of issue it belongs to,
  sweeping the change for other instances, instead of fixing one instance
  per review round.

## [0.1.0] - 2026-06-14

Initial release.

### Added
- `/asymmetric-review:ship` skill: implement → run checks → bounded junior
  review loop → final summary, for any coding task.
- `junior-assumption-reviewer` agent: a read-only reviewer (`Read`, `Grep`,
  `Glob` only) pinned to Sonnet that asks up to 5 assumption-revealing
  questions per round.
- Senior adjudication: every junior question is accepted (smallest useful
  change) or rejected with a stated reason; reworded repeats of settled
  topics are suppressed.
- Convergence stops: the loop ends when a round produces no actionable
  questions, no edits, only cosmetic edits, or only reframes of settled
  topics; the 4-round cap is a backstop.
- Trivial-change skip: pure docs/comment/rename changes bypass the loop.
- Marketplace scaffolding (`.claude-plugin/plugin.json`,
  `.claude-plugin/marketplace.json`) for GitHub installs.
