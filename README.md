# Asymmetric Review

A Claude Code plugin that runs a bounded asymmetric review loop: the main model implements, a read-only junior reviewer on `sonnet` asks assumption-revealing questions, and the main model adjudicates/refines for up to 5 rounds.

## How it works

```
User request
  → Implement code
  → Run checks
  → Fix new failures caused by this work, if any
  → If change is trivial: skip junior review
  → Otherwise [Loop, up to 5 rounds]
      Junior reviewer asks questions (read-only, cannot edit files)
      Senior accepts/rejects each
      Refine + rerun checks if edits made
      Stop when: no actionable questions / no edits made / round cap
  → Final summary (rounds, accept/partial/reject counts, themes, stop reason)
```

The junior reviewer runs on `sonnet` with only `Read`, `Grep`, and `Glob` tools. It cannot modify files. Its role is to surface hidden assumptions, not to prescribe solutions.

## Main session model

Use Opus for the main session. The junior is pinned to `sonnet`, so a Sonnet main session collapses the asymmetry into Sonnet reviewing Sonnet.

## Installation

Install from the GitHub marketplace:

```
/plugin marketplace add arunabh98/asymmetric-review
/plugin install asymmetric-review@asymmetric-review
```

For local development, load the plugin by pointing Claude Code at this repository.

```bash
# From this repository root:
claude --plugin-dir .

# From the parent directory:
claude --plugin-dir ./asymmetric-review
```

## Usage

Inside Claude Code, run:

```
/asymmetric-review:ship implement user profile editing with tests
```

The `$ARGUMENTS` after `/asymmetric-review:ship` can be any coding task:

```
/asymmetric-review:ship fix the off-by-one error in pagination
/asymmetric-review:ship refactor the auth middleware to use async/await
/asymmetric-review:ship debug why the search endpoint returns 500 on empty query
/asymmetric-review:ship refine my current uncommitted changes and add missing edge-case tests
```

If you run `/asymmetric-review:ship` with no arguments, the skill will infer the task from recent conversation and state the inferred task before editing. In a fresh session, or whenever the task is not clear, it should ask instead of guessing.

## Operational notes

The skill tells the model to run checks one at a time, without shell chaining or output truncation. If changed tests clearly use a specific runner or framework, the skill tries that direct check before generic alternatives.

In interactive Claude Code sessions, approve the relevant test/lint/typecheck commands when prompted. In non-interactive `-p` runs, provide appropriate Bash permissions up front if you expect checks to execute; otherwise the skill can still edit and review, but it will report that checks were blocked by permissions.

Check commands may create routine generated artifacts. The workflow uses `git status --short` in git repositories to detect untracked files and distinguish generated noise from implementation changes.

## Verifying the plugin loaded

After launching Claude Code with `--plugin-dir`:

- `/asymmetric-review:ship` should appear as a plugin skill (tab-complete or type it).
- `/agents` should list `junior-assumption-reviewer`, typically referenced from the plugin as `asymmetric-review:junior-assumption-reviewer`.
- `/reload-plugins` picks up edits to plugin files during development.

## Structure

```
asymmetric-review/
  .claude-plugin/
    plugin.json            # Plugin metadata
    marketplace.json       # Marketplace catalog for GitHub installs
  skills/
    ship/
      SKILL.md             # The /asymmetric-review:ship workflow
  agents/
    junior-assumption-reviewer.md   # Read-only junior reviewer subagent
  .gitignore
  LICENSE
  README.md
```

## Limitations (v0)

- Junior review can run up to 5 fresh rounds per ship, with at most 5 questions per round.
- Check commands (test, lint, typecheck) are inferred from the repository; if the correct command is ambiguous, the skill skips the check rather than guessing.
- Does not commit changes unless the user explicitly asks.
