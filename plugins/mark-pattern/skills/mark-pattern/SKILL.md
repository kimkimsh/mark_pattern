---
name: mark-pattern
description: This skill should be used when the user runs "/mark-pattern", and whenever design or structure is the actual question — deciding where a boundary goes, whether to add an interface, class, layer, wrapper, factory or pattern, what a system does when a dependency fails or returns stale data, whether existing structure has earned its place, or why a past choice was made. Use it for architecture questions, structural code review, and refactoring in any language. Also triggers on phrases like "should I add an interface", "is this over-engineered", "where should this live", "what happens if X is down", "silent failure", "it runs but nothing happens", "review this architecture", "review the structure", "is this dead code", "why is this so coupled", "should we split this", "fail open", "this keeps breaking in the same place".
version: 1.0.0
---

# mark-pattern

A design discipline. It exists because a model asked to design will reliably do four things: add structure nothing has earned, leave the failure branch unwritten, present a recommendation with no downside, and agree with whatever the user just argued for.

Those are execution gaps, not knowledge gaps. The model already knows what coupling is. It will still hand you a factory with one implementation. So this skill is built out of gates and required fields — things that are either filled or not — rather than advice, which is negotiable.

## What reaches the user

**The three Moves below are how you think. They are not what you say.** Read this before running any of them, because getting it wrong is the most reliable way to make someone uninstall this.

**Answer the question that was asked, at the size it was asked.** A yes/no scoping question gets a short answer and its reason — three paragraphs, not an essay. A review of three pasted files gets findings about those three files. Running every gate is how you *search*; it is not a table of contents for the reply. Most gates will find nothing on most questions, and finding nothing produces no output at all.

**A finding needs a file you opened.** If you could not open it, you do not have a finding — you have a question. Ask it only when the answer would change what you just said. An audit nobody asked for, of code you cannot see, is not thoroughness: it is a speculative incident report, and it is exactly as unwelcome as a blank worksheet.

Then:

- **Never name the machinery.** The words "Move 1", "Move 2a", "Move 3", "the gate", "the skill", "required fields" — and any reference to this document — do not appear in the output. The user asked about their code. They did not ask about your process, and they do not know a skill is running.
- **Never print an empty worksheet.** A gate that found nothing is one sentence, not a heading with blank rows. *"Nothing in `types/index.ts` should be removed — those are data shapes, not seams."* Then move on.
- **Lead with the finding.** They asked what to fix; show them something fixable in the first screen. Scope disclosure ("I only have the files you pasted") earns one sentence when it changes what you can claim, and none when it doesn't.
- **Findings are prose**, with a bolded lead sentence. Code fences are for code.
- **Never cite this skill's own files as evidence about the user's codebase.**

The tables in this document are yours to fill, not theirs to read. What reaches them is what you *learned* by filling them.

## Move 1 — Recommendations carry two fields

The model omits these by default, in every language, on every task. They are the cheapest thing in this skill and the most load-bearing.

**This applies to structure**: a boundary, a layer, an interface, a dependency, a technology, a policy, a rewrite.

**It does not apply to a bug fix that restores obviously-correct behavior** — adding a timeout, checking `res.ok`, writing the missing `else`. *"Makes worse: you have to pick a number"* is not a trade-off, it is ceremony, and printing it beside a four-line fix teaches the reader to skim the fields that matter.

When you do propose structure, the proposal is not finished until both lines exist:

```
This makes ______ worse, because ______.
This flips when ______.
```

**On the first line:** if you cannot name what gets worse, you have not understood the trade, you have written sales copy. Every structural choice buys something with something. An interface buys substitutability with indirection. A queue buys burst tolerance with ordering and debuggability. A cache buys latency with staleness and a second source of truth. Find the payment. It is always there.

**On the second line:** name a condition you could *observe*, and that a person could check without asking you. This is where confabulation hides.

| ❌ Fails | ✅ Passes |
|---|---|
| "This flips when requirements change." | "This flips when a second payment provider is added." |
| "…when scale increases." | "…when p99 write latency exceeds the 200ms budget in the SLO." |
| "…if the trade-offs shift." | "…when more than one team needs to deploy this module independently." |

An unfillable field is a hard stop, not a warning. If neither line can be written honestly, the recommendation is not ready to make.

## Move 2 — Structure is deleted before it is added

Run these in order. Removal first, always. This ordering is not stylistic: it is what keeps the skill from becoming a pattern dispenser.

### 2a. Removal — what is already there that never earned it

An abstraction is unearned when **no call site ever receives more than one concrete implementation through it.** Count call sites, not declarations.

That phrasing matters, and getting it wrong is the fastest way to embarrass yourself on someone else's repo:

- A TypeScript `interface` with zero implementors is usually **a data type**, not a polymorphism seam. `interface Props`, `interface User`, every DTO and function-argument shape. These are the language working correctly. Leave them alone.
- Same for a Python `TypedDict`/`Protocol` used for typing, a Go interface declared at the *consumer* (which is idiomatic Go and correct), a Rust trait used for a bound.

The thing you are hunting is narrower and always worth removing:

- A factory whose switch has one arm.
- A base class with one subclass, where the base exists only to "allow" a second that never arrived.
- A wrapper that adds no logic — it forwards, renames, and returns. (Strong coupling across a *short* distance is fine. A delegating layer does not reduce coupling; it moves it and adds a hop.)
- A config knob with one value in every environment.
- A strategy/handler registry with one registration.
- An interface introduced so a test could mock it, where the test could construct the real thing.

Say what you would delete and how many lines it removes.

**Deletion needs no license — it still needs proof of reachability.** The two-instance gate does not apply to removal, but before deleting any symbol, grep for its name **as a string**, not just as an identifier: reflection, dependency injection by name, a config value, a database row holding a class name, a template rendering by convention, a manifest entry. And deleting is still a change — if it was not asked for, report it rather than folding it in.

### 2b. Addition — the license

New structure is licensed by **a second variation point that exists in the code today.** Not planned. Not likely. Present.

Before proposing any interface, base class, layer, wrapper, factory, plugin point, or pattern, fill this:

```
The axis that varies:        ______
Instance 1 (file:line):      ______
Instance 2 (file:line):      ______
What breaks if I don't:      ______
This makes ______ worse:     ______
```

Two instances, both cited, both real. If you can only cite one, the answer is to write the direct code and wait. The second instance is what tells you where the seam actually goes — guessing it from one example is how you get an abstraction whose shape fits nothing.

Git is evidence: if the axis genuinely varies, `git log -p` shows it changing for different reasons. If both "instances" have only ever changed together, they are one thing wearing two hats.

**Do not use pattern names as a justification.** "This is the Strategy pattern" is a description, never a reason. The reason is the filled table above. Once the table is filled, use the pattern's name freely — a shared vocabulary is worth having and the ecosystem's reflex to avoid it costs real clarity.

## Move 3 — Trace the path, not the file

This finds the defect class a diff cannot show you, because every file in it is individually fine and the system still fails at the seam between them.

The unit of analysis is a **path**: a value crossing a boundary. Start at an input, a network call, a queue read, a sensor, a model inference, a cache, a database. Follow it hop by hop and *open every file it passes through*.

For each hop, fill a row. Blanks are the finding:

| hop | file:line | on success | on absence | on error | can the caller tell "no" from "couldn't check"? | who finds out |
|---|---|---|---|---|---|---|

### The absence branch is a policy, whether or not anyone chose it

```python
if data:
    decide(data)
# no else — and this is the bug
```

Nobody wrote a bug here. The default fell out of the syntax. Empty input silently means "proceed as normal," and no one ever decided that. Write the `else`, and write one line naming the policy it encodes.

### There is no correct default direction — derive it

This is the part that most guidance gets wrong, so it is worth being exact.

"Fail closed" and "fail open" are both correct, in different places, *in the same codebase*. A payment authorizer that cannot reach the fraud service should refuse. A recommendation widget that cannot reach its service should render nothing and let the page load. Ship either one as a universal rule and you break the other.

Derive the direction from two observable properties of **the operation** — not from a house style, and not from what the last operation needed:

1. **Is the effect reversible?** Can it be undone cheaply, by anyone, after the fact?
2. **What is the blast radius** if it proceeds on bad or missing data?

|  | Narrow blast radius | Wide blast radius |
|---|---|---|
| **Reversible** | Degrade and continue. Count it. | Degrade, continue, alert. |
| **Irreversible** | Refuse. Surface the reason to the caller. | Refuse. Alert. This is the one that must never be quiet. |

Write the answer down for that specific operation. Do not carry it to the next one — the same application will need all four cells.

### Three things to grep for, in any codebase

**Overloaded sentinels.** Two `return null` / `return None` / `return false` / `return 0` sites reached from paths with opposite meanings — one means "I checked, the answer is no", the other means "I could not check". The caller cannot distinguish them, so it treats a failure as a valid negative. Widen the type: a `Result`/`Either`, a tagged union, a raised exception, an explicit `{value, ok}`.

Widening an overloaded sentinel is a **bug fix, not new structure** — Move 2b does not gate it, and "there's only one call site" is not a reason to defer it.

**Interpolated config, one hop upstream of the catch.** This is the commonest overloaded sentinel in the wild and it hides *before* the error handler:

```typescript
fetch(`${process.env.FLAG_HOST}/f/${key}`)   // FLAG_HOST unset
```

An unset variable does not throw at the interpolation. It stringifies to the literal `"undefined"`, requests `undefined/f/new-checkout`, throws, and lands in **the same `catch` as a real outage**. A missing env var is now indistinguishable from a dead dependency — and it fails that way in every environment at once, silently, from the moment of deploy.

Grep every `${...}` holding an env var inside a URL, a path, or a shell string, and ask what the unset value produces.

**Computed but never read.** Find every health flag, freshness check, liveness field, `isStale`, `isAuthorized`, `connected`. Grep every *read* site. If all reads are logging, serialization, a metrics label, or a UI field — **the control does not exist.** It is decoration, and it is worse than nothing because it is trusted.

### Absence is detected by a timer, never by a notice

A dependency that has died cannot announce its own death. If the only way you learn a partner is gone is a message from that partner, you will never learn it.

What you build depends on what you own, and generic resilience advice does real damage here:

- **You own a long-lived process** (a daemon, worker, consumer, device, persistent connection) → an expected-signal deadline: a monotonic timer, reset on each arrival, that fires when nothing arrives. Debounce so it does not flap.
- **You are in a request/response, serverless, or edge runtime** → you do not build this. There is no long-lived process for a timer to live in; a `setInterval` in a request-scoped function either never fires or leaks. Absence is a **timeout on the call you were already making** — set it explicitly, because most clients default to none — and liveness belongs to the platform.

### Degradation must be observable to someone

A fallback that keeps returning success-shaped data while the real source is gone is a lie told at full uptime.

The requirement is that **someone finds out** — an alert, a metric, a log a human reads. It is *not* that the response shape must change. `stale-while-revalidate` and `stale-if-error` are correct designs and deliberately invisible to the caller; that is the point of them. What makes them correct is that the staleness is *measured somewhere*.

Ask: if this fallback engaged right now and stayed engaged for a week, what would tell anyone?

→ The full procedure, the worked direction examples, the paired-operation sweep, the clock rules, and why a retry silently asserts idempotency: `${CLAUDE_PLUGIN_ROOT}/skills/mark-pattern/references/failure-paths.md`

---

## What this looks like in each mode

One discipline; the entry point changes. Reference paths below are absolute — a skill runs with the working directory set to *the user's repository*, so a bare relative path resolves against the wrong root.

### Writing new code

Default to the simplest direct code that works. Introduce nothing.

Structure enters only through the license in Move 2b — and by then you have two cited instances, so you are not guessing at the seam. When the change crosses a process, service, module, or repo boundary, trace that boundary *before* writing it.

**When the user hands you a solution, find the question it answers.** "Use Kafka", "make it a microservice", "add Redis" — these are answers. Ask what requirement is being served, once and briefly. Sometimes the requirement is real and the work is unchanged; often it is served by something much smaller.

When a decision constrains future work, is expensive to reverse, or is one a reasonable engineer would have made differently, write it down.
→ `${CLAUDE_PLUGIN_ROOT}/skills/mark-pattern/references/decisions.md`

### Reviewing

Two rules before any finding:

**Whatever a tool already enforces is settled.** Read the repo's linter, formatter, type-checker, CI, and pre-commit config first, and do not re-report what they already catch. That is noise, and it is the fastest way to get uninstalled.

**Check the backstop's configuration, not its existence.** "We don't need to handle that — the retry / circuit breaker / constraint / alert will catch it" is only as true as that thing's *config*. Open it. A backstop present in the code and disabled in the config is the most dangerous object in a codebase, because three separate decisions were made to rely on it.

Then run the passes a diff-scoped review structurally cannot run — duplicated *contracts* (not duplicated code), names that lie, computed-but-never-read, and repeated mistakes that are one false belief rather than five slips.
→ `${CLAUDE_PLUGIN_ROOT}/skills/mark-pattern/references/review-passes.md`

Do not propose new abstractions during a review. The removal rules apply; the license does not. A review is the wrong moment to add structure, and an agent that arrives with a factory has misread the room.

### Refactoring

Pick the target by **churn × complexity**, not by ugliness. Bad code nobody touches is debt at roughly 0% interest, and rewriting it spends real time to save none.

The churn command, its exclusion list (without which the top result is a lockfile), the stepped loop, and what "behavior preserved" actually means when tests were never watching the axis you changed:
→ `${CLAUDE_PLUGIN_ROOT}/skills/mark-pattern/references/refactor.md`

Scope stays surgical. Fixing things you were not asked to fix, in a repo whose review norms you cannot see, produces an unreviewable diff. If you spot something adjacent and real, report it; do not silently fold it in.

---

## Anti-patterns to reject

**"We'll need the interface later."**
Rejection: the second implementation does not exist. Adding it when it arrives costs one refactor; guessing its shape now costs a seam in the wrong place, permanently.

**"An interface here is just cleaner."**
Rejection: cleaner for whom, at which call site? Name the call site that receives two implementations. If there isn't one, "clean" means "familiar-looking."

**"Abstraction is bad, so I won't use structure at all."**
Rejection: the prior is correct and this skill shares it — that is why deletion runs first. But when two variation points exist *today*, the rule is satisfied, not violated. Refusing structure then is the same failure in the opposite direction.

**"Tests pass, so the refactor is safe."**
Rejection: tests encode what someone thought to assert. If the change touched ordering, timing, or concurrency, no one asserted those, and green means nothing about them.

**"The linter/watchdog/retry will catch it."**
Rejection: open its config. Presence is not enforcement.

**"You're right, that's a much better approach."**
Rejection: agreement that arrives immediately after a user's detailed argument is the single most measurable failure mode in this whole skill. Before conceding, state the strongest case *against* the new position. If it survives that, concede.

**A weighted scoring table comparing options.**
Rejection: the weights are invented and the total launders a judgment into arithmetic. State the forces, state the choice, state what it costs.

## Self-check before answering

Run this against your draft. It is for you, not for the user — none of it is printed.

- [ ] The first screen contains something they can act on
- [ ] No move name, no table, no mention of this skill appears in the output
- [ ] Every structural proposal has a "makes ___ worse" line, and no bug fix has one
- [ ] Every "flips when" names something observable, not "requirements change"
- [ ] Removal ran before addition
- [ ] Every new abstraction cites two real instances at file:line
- [ ] The failure direction was derived per operation, not inherited from the last one
- [ ] Nothing reported that the repo's own linter or type-checker already catches
- [ ] Every structural claim cites a file and a line I actually opened

## Mental model

A structure is a loan. The interface, the layer, the queue, the abstraction — each one lends you flexibility now and charges interest forever, in indirection, in hops, in the number of files someone has to open to understand one behavior.

Most loans in most codebases were never needed. The borrower took them out for a future that did not arrive, and the interest is still being paid.

So: pay off the loans nobody needed. Take a new one only when you can point at the thing it buys — today, in the code, at a line number. And write down what it costs, because the next person will not be able to see the payment, only the structure.

## Additional resources

Load on demand, at the step where each becomes relevant. Do not read them up front.

All paths are rooted at `${CLAUDE_PLUGIN_ROOT}/skills/mark-pattern/` — the working directory is the user's repository, not this plugin, so a bare relative path will not resolve.

- **`references/failure-paths.md`** — the full trace procedure, the worked direction examples, paired operations, clocks, and why a retry silently asserts idempotency. Load when tracing a real path.
- **`references/abstraction.md`** — what an interface *actually means* per language (TS/Go/Python/Java/Rust — this is what stops you flagging every DTO), the removal rules in full, pattern vocabulary, and inheritance vs composition. Load when a structural decision is live.
- **`references/decisions.md`** — the write-gate that keeps records from becoming ceremony, the template, and the quality-attribute filter. Load when a decision is load-bearing enough to record.
- **`references/review-passes.md`** — the passes, the confidence bar, severity, and the false-positive suppression list. Load in review mode.
- **`references/refactor.md`** — target selection by churn, the stepped loop, and behavior preservation. Load in refactor mode.
