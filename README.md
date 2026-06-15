# Asymmetric Review

A Claude Code plugin that has a second, weaker model review your code before you ship it.

Runs entirely in your Claude Code session.

When a model writes code in one pass, it quietly makes choices it never examines (how to handle duplicate inputs, empty lists, odd edge cases) and never flags them. This plugin adds a check. After your main model (the "senior") implements a task, a cheaper, weaker "junior" model reads the change and asks plain, naive questions about it. Those questions tend to surface the assumptions the senior didn't notice it was making, and the senior has to answer each one with a fix or a reason. The reviewer being weaker is the point: it asks the obvious questions the implementer stopped seeing.

## Install

```
/plugin marketplace add arunabh98/asymmetric-review
/plugin install asymmetric-review@asymmetric-review
```

Then, inside Claude Code:

```
/asymmetric-review:ship <your coding task>
```

## Usage

```
/asymmetric-review:ship implement user profile editing with tests
/asymmetric-review:ship fix the off-by-one error in pagination
/asymmetric-review:ship refactor the auth middleware to use async/await
/asymmetric-review:ship refine my current uncommitted changes and add edge-case tests
```

With no arguments, it infers the task from the recent conversation and states it before editing, or asks if the task isn't clear.

## How it works

1. **Implement.** Your main model makes the change and runs the repo's tests, lint, and typecheck.
2. **Review.** A fresh junior reviewer reads the change and asks up to 5 simple questions. It's pinned to a cheaper model (Sonnet) and is read-only: it can look, but it can't edit.
3. **Answer.** Your main model answers every question: accept it and make the smallest useful fix (often a test), or reject it with a reason. Settled questions don't come back.
4. **Repeat or stop.** It reviews again with fresh eyes, and stops as soon as a round turns up nothing new (up to 4 rounds). Trivial changes (pure docs or renames) skip review entirely.
5. **Summarize.** You get a short report: what changed, what the checks said, and what each round accepted or rejected.

It doesn't commit anything; you review the final diff.

## Use a strong main model

Use Opus (or stronger) for your main session. The junior is always Sonnet, so if your main session is also Sonnet you lose the asymmetry: it's just Sonnet reviewing Sonnet.

## What I've seen

In my own testing, the review earns its cost mainly when the first pass is wrong. In one run, the same model that shipped a money-losing bug on its own (silently dropping money from a bill split) caught and fixed that exact bug when run through the plugin. In another, where the first pass was already correct, it found no bugs but added edge-case tests and wrote down an assumption the code had left implicit (that it wasn't thread-safe).

The pattern: **it raises the floor, not the ceiling.** When the first pass is wrong, it tends to catch it; when the first pass is already right, you're paying (a few times the cost) for deeper tests and written-down assumptions. Worth it for code you'll keep; skip it for throwaway scripts. These are observations from a few runs, not a benchmark.

## Local development

`claude --plugin-dir .` from the repo root (or `--plugin-dir ./asymmetric-review` from the parent). `/reload-plugins` picks up edits.

---

MIT · v0.1.0 · Arunabh Ghosh · https://github.com/arunabh98/asymmetric-review
