# Changelog

All notable changes to this plugin are documented here.
Versions follow [semantic versioning](https://semver.org); users receive
plugin updates only when the version in `plugin.json` is bumped.

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
