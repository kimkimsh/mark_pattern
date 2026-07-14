---
name: mark-pattern
description: Turn on the mark-pattern design discipline for this session. With an argument, apply it to that one thing. With "off", turn it back off.
---

Read `${CLAUDE_PLUGIN_ROOT}/skills/mark-pattern/SKILL.md` now, in full, before responding. It holds the gates, the output contract, and the required fields. Do not answer from this file — it deliberately does not restate them, because an agent that answers from a summary never opens the body.

The user's argument was: `$ARGUMENTS`

Dispatch on it:

**Empty argument — turn on session mode.** From here on, design, review, and refactoring answers run under the skill.

Confirm in two or three lines: what is now on, and that they should simply keep working. Do not summarize the skill back at them.

Then take one cheap look at the repo so the first real answer lands: what stack this is, and what is *already enforced* by a linter, formatter, type checker, CI config, or pre-commit hook. Whatever a tool already enforces is settled — the skill does not re-report it. State what you found in one line.

**`off`, `stop`, `그만`, or an equivalent — leave session mode.** Drop back to the normal register and confirm in one line.

**Anything else — treat it as the thing to work on.** Apply the skill to that one request, then stop; do not enter session mode. The request may be a design question, a review, or a refactor; the skill's own mode section decides which path to run.

One reminder, because it is the failure that survives a careless read: **the skill's internal vocabulary never reaches the user.** They asked about their code, not about your process.

Respond in the user's language.
