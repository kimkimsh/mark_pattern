# Failure paths

Load this when tracing a real path. The short form is in SKILL.md Move 3; this is the procedure.

## Why the unit is a path and not a file

The defect this finds has a specific shape: **every file involved is individually fine, passes review, and is written the way a competent person would write it — and the system still fails.** The failure lives in the seam.

A diff-scoped review cannot see it. Neither can a linter, a type checker, or a test that exercises one module. That is not a gap in those tools; it is what "scoped to a file" means.

So you have to walk the path.

## The procedure

**1. Pick the path.** A value that crosses a boundary. Boundaries worth tracing, in rough order of how often they hide this:

- a network call to a service you do not deploy
- a cache read
- a queue or stream consume
- a database read that can return zero rows
- a model or LLM inference feeding a decision
- a config or feature-flag read
- a file/sensor/device read
- an auth or permission check
- anything with a timeout, a retry, or a fallback already attached to it (their presence means someone knew this could fail — and stopped there)

**2. Walk it hop by hop, and open every file.** Not the file you think it is. The file. A name is a hypothesis; `grep` tells you where to look and tells you nothing about what you found.

**3. Fill the table. Blanks are the finding.**

| hop | file:line | on success | on absence | on error | can the caller tell "no" from "couldn't check"? | who finds out |
|---|---|---|---|---|---|---|

Do not summarize instead of filling it. The table's whole value is that omission becomes visible, and prose lets you skip a column without noticing.

**4. For each blank, write the policy that the code currently has by accident**, then decide whether that is the policy you want.

## The absence branch

```python
if data:
    decide(data)
# no else
```

```typescript
const flag = await getFlag("new-checkout");
if (flag) { useNewCheckout(); }
// no else — and "flag service is down" now silently means "old checkout", forever
```

```go
rows, _ := db.Query(...)   // error dropped
for rows.Next() { ... }    // zero rows == "user has no permissions" == "database is unreachable"
```

In each case nobody wrote a bug. The default fell out of the syntax. Someone has to decide what "no data" means, and if no one does, the parser decides.

**Write the else. Then write one line naming what it encodes.**

## Deriving the direction

There is no universally correct direction, and the same codebase will need both. Two properties of the operation decide it:

**Reversibility** — if this proceeds on missing or wrong data, can the effect be undone cheaply, by anyone, afterwards?

**Blast radius** — how far does the damage reach if it proceeds?

|  | Narrow blast radius | Wide blast radius |
|---|---|---|
| **Reversible** | Degrade and continue. Count it. | Degrade, continue, alert. |
| **Irreversible** | Refuse. Surface the reason. | Refuse. Alert. This is the one that must never be quiet. |

Worked, in a single application, showing both answers:

- **Charge a card when the fraud service times out.** Irreversible (a refund is not an undo — it is a second event with its own cost and a support ticket), and wide (a fraud wave). → Refuse. Queue for review.
- **Render the "you might also like" panel when the recommender times out.** Reversible (the next page load fixes it), narrow (one panel). → Render nothing. Increment a counter. Let the page load. Taking down checkout because a recommender blipped is the classic overcorrection.
- **Apply a discount when the pricing service is unreachable.** Irreversible-ish (money out the door), narrow-ish (one order). → Refuse the discount, complete the order at list price, tell the user why.

Same system. Three different directions. Each derived, none inherited.

## Overloaded sentinels

The single most common cross-boundary type defect:

```typescript
// ❌ Both paths return null. The caller cannot tell them apart.
async function getUser(id: string): Promise<User | null> {
  try {
    const res = await fetch(`/users/${id}`);
    if (res.status === 404) return null;   // "this user does not exist"
    return await res.json();
  } catch {
    return null;                            // "the network is down"
  }
}

// caller
const user = await getUser(id);
if (!user) { showEmptyState(); }            // shows "no such user" during an outage
```

```typescript
// ✅ The two answers are two values.
type Fetched<T> =
  | { ok: true; value: T }
  | { ok: true; value: null; reason: "not-found" }
  | { ok: false; reason: "unreachable"; error: unknown };
```

The tell, in any language: **two `return null` / `return None` / `return false` / `return 0` / `return ""` statements in one function, reached from paths that mean opposite things.** Grep for functions with more than one bare falsy return and read why each one fires.

The fix is always the same shape — widen the type so that "I checked, the answer is no" and "I could not check" are different values. `Result`, `Either`, a tagged union, an exception, `{value, ok}`, `(value, err)`. Which one depends on the language. That there must be two is not a matter of taste.

## Computed but never read

A predicate that nothing consumes is not a safeguard. It is decoration, and it is worse than nothing because it is trusted.

Procedure:

1. List every health / freshness / liveness / authorization / staleness field: `isHealthy`, `lastSeenAt`, `connected`, `isStale`, `isAuthorized`, `canEdit`, `heartbeat`, `degraded`.
2. For each, grep **every read site**.
3. Classify each read: **decision** (an `if` that changes what the program does) / **telemetry** (a log line, a metric label, a span attribute) / **presentation** (a UI field, an API response field) / **serialization** (written to a DTO and never looked at again).
4. If a field has zero *decision* reads, that is the finding. Report the read-site table.

```
isAuthorized  →  4 reads:  3 telemetry, 1 presentation, 0 decision   ← the authorization check does not exist
lastHeartbeat →  2 reads:  1 serialization (gRPC field), 1 UI        ← nothing stops when the partner dies
```

The second row is the shape of a real, severe class: a liveness signal written by four producers, read by one consumer, and that consumer paints it on a dashboard. Everyone believes the system knows its partner is alive. Nothing does.

## Absence is detected by a timer

A dead dependency cannot send you a message saying it died. If your only detection mechanism is a message from the thing that failed, you have no detection mechanism.

**What you build depends on what you own.** Generic resilience advice does real damage here.

**If you own a long-lived process** (a daemon, a worker, a consumer, a device, a game loop, a service with a persistent connection):

- an expected-signal deadline: a monotonic timer, reset on each arrival, that fires when nothing arrives within N intervals
- a consumer of that deadline **in the decision path** — not a metric, not a UI field
- debounce, so one late packet does not flap the system
- the timer runs independently of event activity: a heartbeat that is only sent when there is other traffic tells you nothing when traffic stops

**If you are in a request/response, serverless, or edge runtime** (Lambda, Vercel/Next route handlers, Cloudflare Workers, a plain HTTP handler):

- **you do not build this.** There is no long-lived process to hold a timer. A `setInterval` in a request-scoped function either never fires (the runtime freezes between invocations) or leaks (it keeps the instance warm and bills you).
- absence is a **timeout on the call you were already making**. Set it explicitly; most clients default to no timeout, which is an unbounded hang.
- liveness of your *own* service is the platform's job: k8s liveness/readiness probes, load-balancer health checks, uptime monitoring, APM.
- liveness of a *dependency* is a circuit breaker, and a circuit breaker is a timer someone else already wrote.

## Clocks

- **Elapsed time within one process** → monotonic. `performance.now()`, `time.monotonic()`, `std::chrono::steady_clock`, `time.Since()`. Wall clocks jump backwards (NTP correction, leap second, VM pause, a human setting the date) and every duration computed across the jump is silently wrong, often negative.
- **Age of data across services** → you have no shared monotonic clock, and pretending otherwise is worse than using the wall clock. Carry an age computed *at the source* (`{value, observedAt, ttl}`), or compare wall-clock timestamps while explicitly accounting for skew. Do not "fix" a cross-service staleness check into a monotonic comparison — the two clocks are unrelated and the result is nonsense.

## Degradation

The requirement is that **someone finds out** — not that the caller's response shape changes.

`stale-while-revalidate` and `stale-if-error` are correct, standard designs that are *deliberately* invisible to the caller. Bolting a `degraded: true` field onto a public response contract because a design skill told you to is a worse design than the one you started with.

What makes a fallback legitimate:

- [ ] something **counts** it (a metric, a log a human reads, an alert threshold)
- [ ] there is an answer to: *if this engaged right now and stayed engaged for a week, what would tell anyone?*
- [ ] it does not keep making claims the healthy path earns and it no longer can (a cached auth decision may serve a page; it may not authorize a transfer)

What makes it a lie:

- it emits success-shaped output and nothing anywhere increments
- the only signal is a log line at `debug`
- it is indistinguishable, at every observable surface, from the healthy path

## Paired operations

Any operation that opens something must close it on **every** path, including the throwing one.

`subscribe`/`unsubscribe`, `lock`/`unlock`, `open`/`close`, `begin`/`commit`-or-`rollback`, `acquire`/`release`, `addEventListener`/`remove`, `AbortController` created / aborted, `setInterval` / `clearInterval`.

Grep the opener, count it, grep the closer, count it. Unequal counts are a lead, not a verdict — then read the paths. The one that leaks is almost always the error path, because the happy path is the one someone tested.

## Retries import an assumption you did not write down

Adding a retry silently asserts that the operation is **idempotent**. Nobody writes that assertion down, and it is frequently false.

When you see a retry (or add one), answer: *what happens if the first attempt actually succeeded and only the acknowledgement was lost?* If the answer is "the user is charged twice" / "two rows" / "two emails", the retry needs an idempotency key, not a longer backoff.
