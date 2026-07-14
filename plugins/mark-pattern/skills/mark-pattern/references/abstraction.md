# Abstraction

Load this when a structural decision is live. The gate is in SKILL.md Move 2; this is what the gate means in a real language, and how to tell the two failure directions apart.

## The two failures, and why both are real

The ecosystem's prior is **anti-abstraction**, and it is correct: most abstraction in most codebases was never earned. A model prompted about structure will reach for a factory. A model that has read a style guide will refuse structure that has plainly been earned. Both are failures; they just embarrass you in front of different people.

The gate resolves both without arguing with either. Deletion runs first, so the prior is honored. Addition requires two cited instances, so the prior cannot veto structure that has demonstrably been paid for.

## What an "interface" actually means, per language

Getting this wrong is the fastest way to produce a wall of garbage findings. **An interface is not a polymorphism seam in most of the languages people use.**

| Language | Declaration | Usually means | Is it a seam? |
|---|---|---|---|
| TypeScript | `interface User {}`, `type Props = {}` | a **data shape**. Structural typing. Zero implementors is normal and correct. | Almost never. Do not report these. |
| TypeScript | `interface PaymentGateway { charge(): ... }` with `class StripeGateway implements PaymentGateway` | a seam | Yes — now count call sites |
| Go | `type Reader interface` declared **in the consumer package** | idiomatic Go. Accept interfaces, return structs. | Yes, and it is correct even with one implementation today — this is the language's designed style |
| Go | a `MyThingInterface` declared next to its single `MyThing` impl, purely for mocking | mock scaffolding | This is the smell |
| Python | `Protocol`, `TypedDict` | typing | No |
| Python | `ABC` with one concrete subclass | usually the smell | Yes — check call sites |
| Java/C# | `interface IFoo` + `class Foo : IFoo`, one impl, DI-registered | often framework ceremony, sometimes required by the container | Check whether the container or a test is the only consumer |
| Rust | `trait` used as a bound | typing/generics | No |
| Rust | `Box<dyn Trait>` with one implementor | a seam with one side | Yes — check call sites |

**The test that works across all of them:** does any call site, at runtime, actually receive more than one concrete implementation through this? Count call sites. Do not count `implements`.

## Removal rules

Run these first, on the code that exists. Deletion needs no license — but it does need proof of reachability: grep the symbol's name **as a string** before removing it, because reflection, DI-by-name, config values, and template conventions do not show up as identifier references.

**A factory whose switch has one arm.** The factory *is* the constructor, plus a lie about the future.

**A base class with one subclass**, where the base exists to permit a second that never arrived. Ask git: has anything ever been added under it?

**A wrapper with no logic** — it forwards, renames, returns. It did not reduce coupling; it moved it, and added a file someone must now open. Strong coupling at short distance is fine; a delegating layer between two things that always change together is pure cost.

**A config knob with one value in every environment.** Grep the environments. If `FEATURE_X` is `true` in all of them and has been for a year, it is not a knob, it is a fossil, and every branch on it is dead weight.

**A registry with one registration.**

**An interface that exists so a test could mock it**, where the test could construct the real object. The mock is not free: it lets the real object's contract drift out from under the test.

**A layer that every call passes straight through.** If `Controller → Service → Repository` and the Service method bodies are all `return this.repo.x(...)`, the Service is a naming convention, not a layer.

For each: name it, cite it, say how many lines go away.

## Addition — the license, in full

```
The axis that varies:        ______
Instance 1 (file:line):      ______
Instance 2 (file:line):      ______
What breaks if I don't:      ______
This makes ______ worse:     ______
```

**Two instances that exist today.** Not planned, not likely, not "the client said they might want". Present, in the code, at a line number you have opened.

The reason is not purity. It is that **the second instance is what tells you where the seam goes.** With one example you are extrapolating a shape from a single point, and the shape you guess will be wrong in a way that is expensive to correct, because by then things depend on it. With two, the difference between them *is* the axis. The abstraction writes itself.

**Git as evidence.** If an axis genuinely varies, `git log -p -- path/to/thing` shows the two instances changing at different times, for different reasons, in different PRs. If they have only ever changed together, in the same commits, they are one thing with two names — and unifying them behind an interface will make every future change touch three files instead of two.

**The null-parameter test.** If introducing the common interface forces a `null`, an `Optional.empty()`, a `nil`, an unused parameter, or a method that throws `NotSupported` on one side — you have not found a common abstraction. You have found two different things and forced them into one shape. That *adds* complexity. Stop.

## Break-even counts are not gates

You will find break-even heuristics in the literature: a state machine "earns" the State pattern past N states × M actions; a Mediator "earns" itself past four participants. Books state these; they are guesses with a citation.

**Do not let an invented count decide a structural question, and do not print one as though it did.** A number offered as a threshold becomes a rule in the reader's head no matter how you hedge it — which is the same reason a weighted scoring table is rejected. It launders a judgment into arithmetic and makes it unarguable.

Use the counting as a **prompt to go look**, then report what you found, not what you counted.

**This is not a ban on numbers.** A measured number — a p99 latency, an rps figure, a row count, an error rate — is exactly what a "flips when" line is supposed to contain, because it is *observable and checkable by someone else*. That is the opposite of an invented threshold.

| ❌ An invented count deciding a design | ✅ A measured condition that a person can check |
|---|---|
| "You have 9 states × 3 actions, so extract a State machine." | "Flips when sustained writes exceed 500 rps." |
| "Four participants, so introduce a Mediator." | "Flips when a second team needs to deploy this independently." |

## Pattern names

Use them. The ecosystem's reflex against pattern vocabulary costs real clarity — "this is a Circuit Breaker" transmits in three words what a paragraph explains badly, and it lets two people compare trade-offs instead of re-deriving them.

Two rules:

**A pattern name is never a justification.** "This is the Strategy pattern" describes; it does not argue. The argument is the filled license table. Once the table is filled, name the thing.

**A name that does not match the code is worse than no name.** A `FooFactory` that is a builder, a `Manager` that is a bag of unrelated statics, an `EventBus` that has one publisher and one subscriber and could be a function call. When you find a name that lies, the finding is the lie — either rename it or make it true. A reader who trusts the name will make wrong predictions about the code for years.

## Inheritance vs composition

The useful signals, in rough order of how often they decide the question:

**The override test.** If, to inherit, you had to override most of the base's methods — including ones that now throw or no-op — inheritance is wrong. You did not have an "is-a"; you had a "shares some code with", and that is what composition is for.

**Field shadowing.** A derived class declaring a field with the same name as the base's does not override it — it *shadows* it. Now there are two, and which one you get depends on the static type of the reference you happen to be holding. Every language with class inheritance permits this and almost none warn about it. It produces bugs that read as impossible: the same object appears to hold two different values at once.

This is worth a targeted grep on any inheritance hierarchy you touch: list the base's fields, list the derived class's, look for a collision.

**Behavior reuse is not a reason to inherit.** It is a reason to call a function.

**What inheritance is genuinely for:** you need a *subtype* — call sites that hold a base reference and must work, unchanged, with any derived instance. If no call site does that, you wanted composition.

**What you will not use:** the "does it make sense for this object to exist outside its owner?" lifetime question that UML-era texts use to pick composition vs aggregation. In a garbage-collected language nobody makes that decision and nobody can act on the answer. Skip it.

## Two states

**Do not collapse a state to a boolean because there are two of them.** The instinct is backwards, and in modern front-end and async code it is actively harmful.

```typescript
// ❌ permits impossible states: loading && error, data && loading
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState<Error | null>(null);
const [data, setData] = useState<User | null>(null);

// ✅ the impossible states cannot be represented
type State =
  | { status: "idle" }
  | { status: "loading" }
  | { status: "success"; data: User }
  | { status: "error"; error: Error };
```

This is the same rule as the overloaded sentinel in `failure-paths.md`, arriving from the other side: **widen the type until the distinctions you care about are values, and the states you cannot be in cannot be written down.**

Counting states tells you nothing. What tells you something: *can this representation express a state the system can never actually be in?* If yes, it will eventually be in it.
