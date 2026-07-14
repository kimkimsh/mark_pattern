---
name: mark-pattern
description: Turn on the mark-pattern design discipline for the session. Any argument is treated as the first thing to work on — the discipline stays on either way. "off" turns it back off.
---

Read `${CLAUDE_PLUGIN_ROOT}/skills/mark-pattern/SKILL.md` now, in full, before responding. It holds the gates, the output contract, and the required fields. Do not answer from this file — it deliberately does not restate them, because an agent that answers from a summary never opens the body.

The user's argument was: `$ARGUMENTS`

**Invoking this command turns session mode ON and leaves it on.** Design, review, and refactoring answers run under the skill from here until the user turns it off. This holds whether or not an argument was given — an argument is a task, not a scope limit, and the user reaching for this command is asking for the discipline, not for a one-shot.

Dispatch:

**`off`, `stop`, `그만`, `끄기`, or an equivalent — leave session mode.** Drop back to the normal register and confirm in one line. This is the only thing that turns it off.

**No argument — turn it on and stop there.** Confirm in two or three lines: what is now on, and that they should simply keep working. Do not summarize the skill back at them.

Then take one cheap look at the repo so the first real answer lands: what stack this is, and what is *already enforced* by a linter, formatter, type checker, CI config, or pre-commit hook. Whatever a tool already enforces is settled — the skill does not re-report it. State what you found in one line.

**Any other argument — turn it on, then do the work.** Say in one short line that the discipline is on for the session, then answer the request under it. Do not make them ask twice. The request may be a design question, a review, or a refactor; the skill's own mode section decides which path to run.

The argument may arrive wrapped in ordinary prose ("refactor this module with mark-pattern", "리뷰 좀 해줘"). Read through the wrapper to the task. If there is no task in it — it is just a greeting, or the plugin's name repeated — treat it as no argument.

One reminder, because it is the failure that survives a careless read: **the skill's internal vocabulary never reaches the user.** They asked about their code, not about your process.

Respond in the user's language.
