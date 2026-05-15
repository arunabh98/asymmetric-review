# Asymmetric Code

A Claude Code plugin that uses an asymmetric review loop: the main model implements the task, a read-only junior reviewer (running on a cheaper model) asks simple assumption-revealing questions, the main model accepts or rejects those questions, refines the code, reruns checks, and reports the final result.

## How it works

```
User request
  → Implement code
  → Run checks (test / typecheck / lint)
  → Junior reviewer asks up to 5 questions (read-only, cannot edit files)
  → Senior model accepts/rejects each question
  → Refine code if needed
  → Rerun checks
  → Final summary
```

The junior reviewer runs on `haiku` with only `Read`, `Grep`, and `Glob` tools. It cannot modify files. Its role is to surface hidden assumptions, not to prescribe solutions.

## Installation

Load the plugin locally by pointing Claude Code at the plugin directory:

```bash
claude --plugin-dir ./asymmetric-code
```

## Usage

Inside Claude Code, run:

```
/asymmetric-code:ship implement user profile editing with tests
```

The `$ARGUMENTS` after `/asymmetric-code:ship` can be any coding task:

```
/asymmetric-code:ship fix the off-by-one error in pagination
/asymmetric-code:ship refactor the auth middleware to use async/await
/asymmetric-code:ship debug why the search endpoint returns 500 on empty query
/asymmetric-code:ship refine my current uncommitted changes and add missing edge-case tests
```

## Verifying the plugin loaded

After launching Claude Code with `--plugin-dir`:

- `/asymmetric-code:ship` should appear as a plugin skill (tab-complete or type it).
- `/agents` should list `junior-assumption-reviewer`.
- `/reload-plugins` picks up edits to plugin files during development.

## Structure

```
asymmetric-code/
  .claude-plugin/
    plugin.json            # Plugin metadata
  skills/
    ship/
      SKILL.md             # The /asymmetric-code:ship workflow
  agents/
    junior-assumption-reviewer.md   # Read-only junior reviewer subagent
  README.md
```

## Limitations (v0)

- The junior reviewer asks at most 5 questions and runs read-only with `Read`, `Grep`, and `Glob`.
- Check commands (test, lint, typecheck) are inferred from the repository; if the correct command is ambiguous, the skill skips the check rather than guessing.
- Does not commit changes unless the user explicitly asks.
