# mark-pattern

**A design discipline for Claude Code.** One command turns it on; from then on, Claude designs, reviews, and refactors under it.

It exists because a coding agent asked to design will reliably do four things:

1. add structure that nothing has earned,
2. leave the failure branch unwritten,
3. present a recommendation with no downside, and
4. agree with whatever you just argued for.

Those are not knowledge gaps. The model knows what coupling is. It will still hand you a factory with one implementation. **mark-pattern is built out of gates and required fields — things that are either filled or not — rather than advice, which is negotiable.**

Language-neutral by design. Tested on TypeScript, React, Go, and Python.

---

## Install

```bash
/plugin marketplace add kimkimsh/mark_pattern
/plugin install mark-pattern@mark-pattern
```

Then restart Claude Code (or start a new session).

<details>
<summary>Installing from a local clone instead</summary>

```bash
git clone https://github.com/kimkimsh/mark_pattern.git
```

Then, in Claude Code:

```bash
/plugin marketplace add /absolute/path/to/mark_pattern
/plugin install mark-pattern@mark-pattern
```

Useful if you want to edit the skill and try your changes immediately.
</details>

<details>
<summary>Verifying it installed</summary>

```bash
/plugin              # mark-pattern should be listed and enabled
```

If `/mark-pattern` is not recognized, the plugin loaded but the session predates it — start a new session.
</details>

---

## Use

| Command | What happens |
|---|---|
| `/mark-pattern` | Turns the discipline on for the session. Then just work — it applies as you go. |
| `/mark-pattern <task>` | Applies it to that one thing. Session mode stays off. |
| `/mark-pattern off` | Back to normal. |

The skill also activates on its own when design is the real question — *"should I add an interface here"*, *"is this over-engineered"*, *"what happens if that service is down"*, *"why does this keep breaking in the same place"*.

**There is one skill and one command.** You do not pick a sub-skill, and you do not tell it which mode you want. It reads what you asked for.

Once it is on, you code, review, and refactor the way you normally would. The difference shows up in what Claude does *before* it writes:

```
you  ▸ /mark-pattern
     ◂ On. This is a Next.js/TS repo; eslint + tsc + prettier run in CI,
       so I won't re-report anything they already catch.

you  ▸ add a notification service — email now, SMS later
     ◂ Writing the email path directly. No NotificationProvider interface yet:
       the second implementation doesn't exist, and the second one is what tells
       you where the seam actually goes. When SMS lands, the difference between
       them *is* the abstraction — guessing it now from one example gets the
       shape wrong in a way that's expensive to correct later.

       One question before I write. When the email provider is unreachable, what
       should happen? A password-reset mail and a marketing blast want opposite
       answers here — the first is irreversible from the user's side and should
       fail loudly; the second should queue and move on.
```

---

## What it actually does

Three moves, in this order. The order is not stylistic.

### 1. Every recommendation carries two fields

```
This makes ______ worse, because ______.
This flips when ______.
```

The model omits both by default, in every language, on every task. They are the cheapest thing in the plugin and the most load-bearing.

The first forces the trade into the open. An interface buys substitutability with indirection. A queue buys burst tolerance with ordering and debuggability. A cache buys latency with staleness and a second source of truth. **If you cannot name what gets worse, you have written sales copy, not a recommendation.**

The second has to name something you could *observe* — and this is where models confabulate:

| ❌ Rejected | ✅ Accepted |
|---|---|
| "flips when requirements change" | "flips when a second payment provider is added" |
| "flips when we need to scale" | "flips when sustained writes exceed 500 rps" |

An unfillable field is a hard stop, not a warning.

### 2. Structure gets deleted before it gets added

**Deletion runs first.** This is what keeps the plugin from becoming a pattern dispenser.

It hunts factories whose switch has one arm, base classes with one subclass, wrappers that forward and return, config knobs with one value in every environment, and interfaces that exist only so a test could mock something it could have constructed for real.

**Then, and only then, addition** — licensed by *a second variation point that exists in the code today*:

```
The axis that varies:        ______
Instance 1 (file:line):      ______
Instance 2 (file:line):      ______
```

Two instances, both cited, both real. Not planned. Not likely. Present.

> **On the ecosystem's anti-abstraction reflex.** Every other guidance skill points one way: no abstraction for a single use site. That prior is *correct*, and mark-pattern shares it — which is exactly why deletion runs first. But when two variation points genuinely exist, the rule has been *satisfied*, not violated, and refusing structure at that point is the same failure in the opposite direction. mark-pattern is the only thing in the ecosystem that says what to do when the second implementation actually arrives.

### 3. The unit of analysis is a path, not a file

This finds the defect class where **every file is individually correct, passes review, and the system still fails at the seam between them.** A diff cannot show you this. Neither can a linter or a type checker. That is not a gap in those tools — it is what "scoped to a file" means.

So it walks the path, opens every file, and fills a row per hop:

| hop | file:line | on success | on absence | on error | can the caller tell "no" from "couldn't check"? | who finds out |
|---|---|---|---|---|---|---|

Blanks are the finding. Three things it reliably catches:

**The absence branch nobody chose.**

```python
if data:
    decide(data)
# no else — and the parser just made a policy decision on your behalf
```

**Overloaded sentinels.** Two `return null` sites reached from paths that mean opposite things — one means *"I checked, the answer is no"*, the other means *"I could not check"*. The caller cannot tell them apart, so it treats an outage as a valid negative answer.

**Computed but never read.** An `isAuthorized` / `isHealthy` / `lastSeenAt` field whose every read site turns out to be a log line, a metric label, or a dashboard. **The control does not exist. It is decoration** — and it is worse than nothing, because it is trusted.

---

## What it deliberately does not do

Restraint is most of the design here.

**It has no default failure direction.** "Fail closed" and "fail open" are both correct, in different places, *in the same codebase*. A payment authorizer that cannot reach the fraud service should refuse. A recommendation widget that cannot reach its service should render nothing and let the page load. Ship either as a universal rule and you break the other — so mark-pattern derives the direction per operation, from two observable properties: **is the effect reversible, and what is the blast radius?**

**It does not re-report what your tools already catch.** The first thing it does is read your linter, formatter, type-checker, and CI config. Whatever they enforce is settled.

**It does not recommend patterns.** A pattern name is a *description*, never a justification. Once the license table is filled, it will happily call the thing a Circuit Breaker — a shared vocabulary is worth having, and the ecosystem's reflex against it costs real clarity. But the name never carries the argument.

**It does not add ceremony.** Most decisions get no decision record. One is written only when the choice constrains future work, is expensive to reverse, or a reasonable engineer would have chosen differently. *"We used the standard library's JSON parser"* is not a decision. *"We wrote our own"* is.

**It does not compete with your other skills.** It is a lens, not a gate. It never claims you must run it before anything else, so it does not fight `superpowers`, `karpathy-guidelines`, `/code-review`, or `/simplify` for control of a turn.

**It does not touch style.** Naming, formatting, comment density, magic numbers, function length — every linter on earth already owns those, and restating a hard, deterministic check in a soft, advisory medium only dilutes it.

---

## Where it comes from

This is distilled from a long research effort, not from a blog post.

- **Five books, read cover to cover** (~3,100 pages) — software architecture, design patterns, OOA&D, clean code, and process — including the places where they contradict each other, and the places where they have aged badly.
- **Six production repositories, ~408k lines**, read at source level across C++, Java, Python, and firmware. Rules here were earned by finding real defects, not deduced from a book.
- **A full audit of all 363 installed Claude Code skills**, which is how the gap this fills was identified. Not one of them mentions a single design pattern by name. The only stance the ecosystem takes on structure is a uniform, correct, and entirely unconditioned anti-abstraction reflex — nobody says what to do when the second implementation arrives, nobody says how to decide where a boundary goes, and nobody records why.
- **Research into how coding agents actually fail at design** and, separately, into what makes a skill *work* at runtime instead of getting ignored.

That last one shaped the form as much as the content. Three findings changed how this is written:

> **Prohibitions backfire.** In head-to-head tests, a "don't do X" instruction produced *more* of the unwanted output than a positive recipe — and trended worse than no guidance at all. Agents negotiate with "don't". A recipe leaves nothing to negotiate. So this plugin is built from required fields and filled tables, not from a list of bans.

> **"Keep it simple" does not work.** Karpathy, on trying to get models to simplify his own code: *"The models hate this. They can't do it. It feels like pulling teeth."* Simplicity has to be structurally forced. Hence: deletion runs first, and every addition needs two citations.

> **Shouting does not help.** Current models follow the system prompt more literally, so `CRITICAL: YOU MUST` now causes *over*triggering rather than compliance. There is none of it in here.

The research corpus itself is not published — it contains analysis of a private codebase. Every example in the plugin is synthesized and language-neutral.

---

## FAQ

**Will it flag every `interface` in my TypeScript repo?**
No — and getting that wrong was the loudest finding in testing. A TS `interface` with no implementors is a **data type**, not a polymorphism seam: `interface Props`, every DTO, every function-argument shape. The plugin counts *call sites that receive more than one concrete implementation*, never `implements` declarations. Same for `Protocol` and `TypedDict` in Python, and for a Go interface declared at the consumer — which is idiomatic and correct even with one implementation today.

**Does this work outside TypeScript, Python, and Go?**
It should, and it has not been measured there — so here is the honest split. The plugin was live-fire tested against **TypeScript, React, Go, and Python**, each with a no-guidance control. Those four are where the evidence is. Every rule is about structure rather than syntax, and the one place a language's *semantics* actually change the answer — what an `interface` declaration means — is handled explicitly per language, so the design is meant to carry. But "meant to carry" is a claim about the design, not a measurement, and you should read it that way.

**Will it fight my existing skills?**
No. It makes no claim to run first and registers no gate. If you have `superpowers` installed, `brainstorming` still owns the pre-work gate; mark-pattern is the design *method* that runs while you are inside it.

**Is "no findings" an acceptable answer?**
Yes, and it will say so rather than manufacturing a tenth item. Findings are capped at ten and reported only above a real confidence bar.

**Does it write files into my repo?**
Only decision records, only when a decision clears the write-gate, and it proposes the path first.

---

## License

MIT
