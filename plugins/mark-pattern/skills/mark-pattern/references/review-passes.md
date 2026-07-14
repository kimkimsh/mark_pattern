# Review passes

Load this in review mode. These are the passes a diff-scoped review structurally cannot run — not because reviewers are careless, but because the evidence is not in the diff.

## Before any finding

**1. Read what is already enforced.** The repo's linter, formatter, type-checker, CI workflow, pre-commit hooks, and `CONTRIBUTING`/`AGENTS`/`CLAUDE` files.

Everything a tool already enforces is **settled**. Do not re-report it. A review that opens with findings the user's own linter catches on save is noise, and it is the fastest way to get this uninstalled. If a rule matters and no tool enforces it, that is itself a finding — propose the check, in the project's own stack.

**2. Find the reference implementation, not the median file.** The default instruction every coding agent carries — "follow existing conventions" — is a trap in a codebase where nine modules out of ten are the disease and one is the cure. Identify the newest, best-structured, actually-tested module and treat *that* as the convention. When the majority pattern and the newest pattern disagree, say so out loud rather than silently averaging them.

**3. Set the bar.** Report a finding only at ≥80% confidence that it is real *and* worth the reader's time. Cap the list at ten and rank by consequence. **"No structural findings" is a legitimate and expected result** — say it plainly rather than manufacturing a tenth item. Design findings are more subjective than bug findings, which means the threshold matters more here, not less.

## The passes

### Pass 1 — Directory names are claims, not evidence

Print the tree to depth 2–3 (exclude `node_modules`, `vendor`, `dist`, `build`, `target`, `.venv`). For each top-level component, get LOC and file count. Then check the name against the contents.

The finding is a **name that lies**: a `core/` holding 400 lines while `utils/` holds 12,000; a `util/` that turns out to hold the domain model; a `common/` that everything imports and nothing owns. When the load-bearing code lives somewhere its name disclaims, every newcomer looks in the wrong place, and the code accumulates there precisely *because* nobody is watching it.

### Pass 2 — Duplicated contracts, not duplicated code

Copy-paste detectors find duplicated *code*. They do not find a duplicated *contract*, and the contract is the one that hurts.

The shape: two modules — often in different repos, sometimes in different languages — that must agree on a wire format, an enum, a set of opcodes, a status code, a field ordering. They share zero lines. A similarity detector scores the codebase clean. They are synchronized by a comment that says *"keep this identical to X"*, or by nothing at all.

**Both sides still compile after one of them diverges.** That is the entire problem. There is no build error, no type error, no failing test — just a runtime mismatch in production, weeks later.

Grep for: parallel enum/constant blocks in two places, matching magic numbers across a boundary, a comment containing "keep in sync" / "must match" / "mirror of" / "same as", hand-written serializers on both ends of a wire.

The finding is not "these are duplicated." It is **"nothing makes these fail together."** Ask where the generator is — and if the answer is that there isn't one, that is the recommendation.

### Pass 3 — The backstop's configuration

The most dangerous object in a codebase is a safeguard that is present in the code and disabled in the config, because *three separate decisions were made to rely on it*.

Any time the code, a comment, a PR, or the user says *"we don't need to handle that — the retry / watchdog / circuit breaker / DB constraint / rollback / alert will catch it"*:

**Open the config, not the code.** Is the retry actually configured with a nonzero count? Is the watchdog's reboot flag set, or merely the watchdog compiled in? Does the constraint exist in the migration, or only in the model? Is the alert routed to a channel a human reads, or to a channel muted in 2023?

The backstop being *implemented* proves nothing. It has to be *turned on*, and the two facts live in different files, which is why nobody checks.

### Pass 4 — Computed but never read

Run the read-site classification from `failure-paths.md`. A health, freshness, liveness, or authorization predicate whose every read site is logging, serialization, a metric label, or a UI field **is not a control.** Report the read-site table.

### Pass 5 — The path, not the file

Run Move 3 for every boundary the reviewed code crosses. This is the pass that finds the class of defect where every file is individually correct.

### Pass 6 — Repeated mistakes are one false belief

If the same kind of error appears in five places, do not file five findings. **Consistency is evidence of a systemic cause, not of five independent slips.** Somebody believed something false — about how an API behaves, about what a function returns, about which thread runs what — and acted on it consistently.

Find the belief. Fix the belief. Then the five sites are one finding with five instances, which is a completely different conversation, and a much cheaper fix.

The same applies to a bug that resists three fixes: three failed fixes means the model of the system is wrong, not that the fourth patch will land. Stop patching and go find what you believe that isn't true.

### Pass 7 — Structure that never earned its place

Run the removal rules from `abstraction.md`. Do **not** propose new abstractions during a review — a review is the wrong moment to add structure, and an agent that arrives with a factory has misread the room.

## False positives — do not report these

- Anything the repo's linter, formatter, or type-checker already enforces.
- Style, naming, or formatting, unless a name actively lies about what the code does.
- Comment density, magic numbers, function length. Length in particular is not a defect; it is a symptom that may or may not have one behind it, and *"this function is 60 lines"* is not a finding.
- Missing tests, stated as a bare recommendation. Check what verification the repo actually has and can run before prescribing any. "Add tests" is unactionable in a repo with no harness, and it reads as a reflex rather than a review.
- Test code held to production structure rules. Duplication in tests is often correct — each test should be readable alone.
- Generated code, vendored code, lockfiles, migrations, `dist/`.
- A "problem" you inferred from a name without opening the file.
- Anything phrased as "consider", "you might want to", or "it could be argued". If it does not clear the bar, cut it. Hedged findings are padding, and they train the reader to skim.

## Severity

Derive it from **this** repo's consequences, not from a table:

- What here is **irreversible**? (money moved, data deleted, a message sent, a key leaked)
- What here is **expensive to detect**? A wrong number in a report may sit for a year. A crash is found in a minute. The quiet failure often outranks the loud one.
- What here is **load-bearing**? A defect in something imported by 200 files is not the same defect as one in a leaf.

Never import a severity threshold from another domain. A twenty-second stale reading is a catastrophe in one system and entirely unremarkable in another, and nothing about the code tells you which — only the consequences do.

## The output

**Lead with the finding.** The user asked what to fix — they must see something fixable on the first screen. Any framing (what you could not open, the trace itself, the single false belief behind several bugs) earns at most three sentences and goes *after* the top finding, or gets cut. The pass structure above is how you searched; it is not how you report.

Each finding is prose: a bolded lead sentence saying what is wrong, then the supporting lines. **Do not wrap findings in a code fence** — prose in a fence renders as non-wrapping monospace and makes the most important content on the page harder to read. Fences are for code.

> **`getFlag()` returns `false` both when a flag is off and when the flag service is unreachable.**
> `lib/flags.ts:6` — the `catch` swallows every failure into the same value the caller reads as "feature disabled".
> **Why it bites:** if `FLAG_HOST` is misconfigured or the service is down, every user silently gets every legacy code path, indefinitely. Nothing counts it, nothing alerts, and the app looks healthy.
> **Fix:** make the caller name its fallback — `getFlag(key, fallbackWhenUnavailable)` — so the choice is written down at the site that knows what it means.
> **Cost:** every call site now has to state a default. That is the point: it is currently being chosen by a `catch` block.

A finding that cannot be given a file and a line you actually opened is a hypothesis. Go open the file, or drop it.
