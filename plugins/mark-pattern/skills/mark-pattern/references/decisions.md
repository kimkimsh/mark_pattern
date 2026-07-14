# Decision records

Load this when a decision is load-bearing enough to record. The two required fields are in SKILL.md Move 1; this is the record they go in, and — more importantly — the gate that stops this from becoming ceremony.

## The write-gate: most decisions get no record

A record for every decision is how this practice dies. The team writes forty documents, nobody reads them, and the habit is abandoned inside a quarter — and the abandonment is *correct*, because forty documents that record nothing worth recording are worse than none.

Write a record only when at least one is true:

- [ ] It **constrains future changes.** Undoing it later means touching code that does not exist yet.
- [ ] It is **expensive to reverse.** Data has to be migrated, a contract has to be renegotiated, a dependency has to be ripped out.
- [ ] **A reasonable engineer would have chosen differently.** If every competent person picks the same option, there is nothing to record — the option *is* the default.
- [ ] It is being made **for a non-obvious reason.** The obvious choice was rejected, and in six months nobody will remember why.

Fails all four → no record. Say so and move on. "We used the standard library's JSON parser" is not a decision. "We wrote our own JSON parser" is.

## The template

Short. Fits on a screen. Lives in the repo, in `docs/decisions/` unless the repo already has somewhere better — check for an existing `adr/`, `rfc/`, `docs/architecture/` and use it rather than founding a rival.

```markdown
# NNNN — <the decision, as a sentence in the active voice>

**Status:** Proposed | Accepted | Superseded by NNNN
**Date:** YYYY-MM-DD

## Context
What forces are in play. What is true now that makes this a question at all.
No solutions in this section.

## Options
Two or more, honestly stated. An option written only to lose is not an option;
it is a rhetorical device, and a reader will spot it.

## Decision
What was chosen.

## Consequences
What this buys.

**This makes ______ worse, because ______.**    <- required. No decision has no downside.
                                                   If you cannot find one, keep looking.

**This flips when ______.**                     <- required. An OBSERVABLE condition.
                                                   Not "requirements change".

## Confirmation
How anyone can tell this is still being followed. A test, a CI check, a lint rule,
a query, a dashboard panel. If nothing automated is possible, say so, say what the
manual check is, and say why automation was impossible.
```

Immutable once accepted. Superseding a record means writing a new one and linking both ways — never editing the old one. The value of the record is that it captures what was known *at the time*, and an edited record has lost exactly that.

## What the fields are defending against

**"Consequences" exists because a model presents recommendations with no downside.** Left to itself it produces a bulleted list of benefits — which is sales copy, and it is unfalsifiable, and it teaches the reader nothing. The forced negative is what converts a pitch into a decision.

If you genuinely cannot find a downside, you have not understood the option well enough to choose it. Every structural choice pays for something. Find the payment.

**"Flips when" exists because conditions change and nobody revisits.** It is the difference between a decision and a dogma. Without it, a choice that was right for a two-person prototype governs a forty-person system three years later, and no one can say when it stopped being right.

It also has a confabulation failure that is worth naming, because it is the most likely way this whole practice quietly becomes worthless:

| ❌ Fails | Why | ✅ Passes |
|---|---|---|
| "…when requirements change" | Not observable. Always true. Never checkable. | "…when a second tenant needs isolated data." |
| "…when we need to scale" | Which number, past what? | "…when the write path exceeds 500 rps sustained." |
| "…when the trade-offs shift" | Restates the question. | "…when the vendor's SLA drops below 99.9% or they raise the per-seat price above $X." |

The test: **could someone else check whether it has happened, without asking you?** If not, rewrite it.

**"Confirmation" exists because decisions decay silently.** A rule nobody checks is a rule nobody follows, and the first person to break it will not know they did. This is the section that most often reveals the decision was never real: if there is no way to tell whether the codebase still follows it, it was a preference, not a decision.

Degrade gracefully. Many repos have no CI. "No automated check is possible here; the manual check is *X*; it is manual because *Y*" is a legitimate and honest answer, and it is far better than inventing a CI job that does not exist.

## No scoring tables

```
❌  Option    Perf  Cost  Maint  Weight  Total
    A         8     6     7      x1.2    25.2
    B         5     9     8      x1.0    22.0
```

Reject this on sight, including — especially — when you produced it yourself.

The weights were invented. The scores were invented. The arithmetic is real, and that is the problem: it converts a judgment into a number, and a number is very hard to argue with. It launders the choice you had already made into something that looks like a calculation. Anyone who disagrees now has to attack the arithmetic instead of the reasoning, which is exactly the wrong conversation.

State the forces. State the choice. State what it costs. Let it be arguable.

## Quality attributes: pick at most three, and only measurable ones

If you are choosing between structures, you need to know what the structure is *for*. "It should be scalable, maintainable, secure, performant, and reliable" is not a specification — it is a list of every good word, and it selects nothing, because every option scores well on a word.

Two filters:

**Measurable.** "Fast" is not a requirement; "p99 under 200ms at 1000 rps" is. If you cannot state how you would *tell*, it cannot drive a decision, because no option can be shown to satisfy it.

**Critical to success.** Not "nice". If this attribute were absent, would the project fail? Most attributes on most projects would not, and admitting that is what lets the two or three that would actually decide something.

At most three. Past three, everything is a priority and therefore nothing is — and the model will happily agree that all five matter, which is precisely why the cap is a number and not a suggestion.

## Do not let the solution smuggle in the requirement

A specific request — "use Kafka", "make it a microservice", "add Redis" — is an *answer*. Before implementing it, find the question.

Ask what requirement it serves. Then check whether it is the best way to serve it, and whether the requirement is real. Sometimes it is, and the work is unchanged. Often the requirement is served by something much smaller, and occasionally the requirement does not exist at all — the technology arrived from a conference talk.

This is one of the few places where the right move is to ask the user a question rather than produce output. Ask once, briefly, and proceed.
