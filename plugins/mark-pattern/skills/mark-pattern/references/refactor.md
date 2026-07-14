# Refactoring

Load this in refactor mode. Refactoring changes structure without changing behavior. The moment behavior changes, it is not a refactor any more — it is a change, and it needs the review a change gets.

## Pick the target by churn, not by ugliness

Bad code nobody touches is debt at roughly 0% interest. It is not costing anything, and rewriting it spends real time to save none.

The code worth restructuring is the code that is **both** complex **and** changed constantly — every edit pays the complexity tax again, and that tax is the interest.

```bash
git log --since=1.year --name-only --pretty=format: \
  | grep -Ev '(^$|lock|\.min\.|/dist/|/build/|/vendor/|/node_modules/|/target/|\.generated\.|_pb2?\.|\.pb\.go|\.snap$|/migrations/)' \
  | sort | uniq -c | sort -rn | head -20
```

**The exclusions are load-bearing.** Without them the top result on any JS repo is `pnpm-lock.yaml`, on any Go repo it is a generated protobuf, and the first thing the user sees is you nominating a lockfile for refactoring. Adapt the pattern to the repo you are actually in — check what is generated there before you run it.

Cross the churn list against complexity (nesting depth, branch count, file length, number of things imported). The intersection is the list. Everything else waits, and saying so is part of the answer: **"these three files are worth it; the rest of the codebase is ugly and cheap to leave alone"** is a better outcome than a fifty-item backlog nobody will ever burn down.

## The loop

Small enough to verify between. Not a phase — a loop, and you run it once per step.

**1. Pin the current behavior.**

Before changing anything, know what "unchanged" means. In order of preference:

- The tests that already exist, **observed passing right now**. Run them. Do not assume.
- A characterization test, if none exist: a test that asserts what the code *currently does*, whether or not that is what it *should* do. This is not a test of correctness. It is a tripwire. Write it around the seam you are about to touch, watch it pass, and keep it until the refactor lands.
- If the code cannot be tested at all: that is the finding, and the first refactor is whatever makes it testable — usually breaking one hard-coded dependency. Say this rather than proceeding blind.

**2. Make one structural change.**

One. Rename, or extract, or inline, or move — not all four. The reason is not tidiness: it is that when the tests go red, you need to know which change did it, and a four-part step tells you nothing.

**3. Verify.**

Run the tests. Run the type checker. Run the build. Then look at what you actually changed.

**4. Repeat, or stop.**

Stop when the churn-weighted pain is gone, not when the code is beautiful. "Beautiful" has no stopping condition, and a model asked to keep improving will keep going forever.

## Behavior preservation

**Green tests are not proof.** Tests encode what someone thought to assert. If your change touched **ordering, timing, or concurrency**, nobody asserted those — the tests were never watching that axis, and they will stay green through a real regression.

Concretely, the refactors where green means nothing:

- replacing a blocking wait with an event or a callback (the blocking wait had an *implicit observation window* that nothing in the tests names, and removing it can drop the thing that was being observed)
- changing when a lock is taken or released
- reordering initialization
- making something async that was sync, or vice versa
- collapsing two loops into one, when something else was observing the state between them

For those, you need reasoning, not a green checkmark. Say what the old code observed and when, and show that the new code still observes it.

**Two commits, always.** A refactor commit and a behavior commit are different objects, and mixing them makes both unreviewable — a reviewer cannot tell which of your 400 changed lines was the fix and which was tidying, so they review neither.

If a refactor and a fix must both happen: refactor first, verify green, commit. Then fix, and let the fix be a small diff against clean structure. That ordering also makes the fix easier to write.

## The traps

**Merging two blocks that merely look alike.** Two functions with the same shape and different reasons to change are not duplication. Unify them and the next requirement forces a flag parameter, then a second flag, and now one function serves two masters and both are worse. Before unifying: check git. Have they ever changed in the same commit, for the same reason? If they have only ever changed apart, they are not the same thing.

**Deleting something reachable by a path you did not grep.** Reflection, dependency injection by name, a string in a config, a database row holding a class name, dynamic dispatch, a template rendering by convention, an entry point declared in a manifest. Before deleting a symbol that looks orphaned, grep for its name as a **string**, not just as an identifier.

**Folding in an unrequested cleanup.** You are working in someone's repo, with review norms and diff-size culture you cannot see. An unrequested 600-line "while I was in there" is the single fastest way to have good work rejected. Report what you spotted; let them ask.

**Extracting until the pieces are meaningless.** A split is good only when the resulting interface is *simpler than the implementation it hides*. If a reader has to flip between the two halves to understand either one, the split made things worse — you have doubled the number of things to understand and hidden nothing.

Line count is not the objective. Neither is function length. The objective is that a reader can hold one piece in their head at a time.

## When to stop and say so

Refactoring is not always the answer, and a model asked to refactor will refactor.

If the structure is wrong because the **model of the domain** is wrong — the entities are cut in the wrong place, an identity is keyed on something mutable, two concepts are fused into one — no amount of moving code will fix it. Rearranging the pieces of a wrong model produces a tidier wrong model.

Say that. It is a design finding, and it belongs in a decision record, not in a diff.
