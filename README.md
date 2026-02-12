# Discussion: Abort Protocol Discussion

## Status

No Status - This is a discussion, not a proposal

## Champions

TBD

## Motivation

Cancellation is a fundamental concern of asynchronous and long-running programming.
The JavaScript ecosystem has converged on the Web API `AbortController`/`AbortSignal`
as the *de facto* cancellation mechanism, with adoption across `fetch()`, Node.js
streams, database drivers, and countless libraries.

However, `AbortController` and `AbortSignal` are defined in the **DOM specification**
(WHATWG), not in ECMAScript. TC39 cannot normatively reference Web APIs, and not all
JavaScript environments are required to implement them. This creates a gap: the
language has no built-in concept of cancellation, even though the entire ecosystem
depends on one.

This document discusses the design principles for an **abort protocol** — a minimal set
of requirements that any object can satisfy to participate in cooperative cancellation.

## Design Principles

### Principle 1: A Mechanism for Receiving an Abort Signal

The core requirement is a way for a **consumer** of work to be told "stop what you're
doing." This is a one-way communication: from a controller (the party that decides to
cancel) to a consumer (the party doing cancellable work).

The consumer needs a way to register interest in being notified when abort occurs.

### Principle 2: Synchronous and Immediate Notification

When an abort is triggered, registered listeners **must** be invoked synchronously — in
the same turn, with no microtask or macrotask deferral. This is non-negotiable because:

- The consumer may be about to commit a side effect (write to disk, send a network
  request, start a subprocess). Any async deferral means the side effect proceeds
  anyway, defeating the purpose of cancellation.
- Cancellation is inherently time-sensitive. If a consumer is between iterations of a
  `for await` loop, the abort notification must fire *now*, not after the next promise
  settles.
- This matches existing `AbortSignal` behavior, where `abort` event listeners fire
  synchronously when `controller.abort()` is called.

Synchronous notification is what distinguishes an abort signal from a mere "please stop
when convenient" hint. It is an interrupt, not a suggestion.

### Principle 3: Pre-Check — Has Abort Already Occurred?

Before starting work, a consumer must be able to answer: **"Has this signal already
been aborted?"**

This handles the case where:
- The abort occurred before the consumer received the signal object
- A function is called with an already-aborted signal due to timing or control flow
- Work is queued and by the time it dequeues, cancellation has already happened

Without a pre-check, consumers would have to register a listener and then immediately
check if it already fired, which is error-prone and awkward.

```js
async function doWork(signal) {
  // Without pre-check, this is a race:
  // what if abort happened before we got here?
  
  // With pre-check:
  if (signal.aborted) throw signal.reason;
  
  // Safe to proceed
}
```

### Principle 4: Post-Check — Did Abort Occur While I Wasn't Listening?

There are windows where a consumer is performing synchronous or otherwise
uninterruptible work and **cannot actively respond** to an abort notification when it
fires. After such a window, the consumer must be able to determine: "Did an abort
happen while I was busy?"

This is the same mechanism as the pre-check — a state property that reflects whether
abort has occurred. But it serves a distinct use case:

```
1. Check state      → not aborted
2. Register listener
3. Begin work
4. Enter synchronous computation (listener cannot meaningfully respond mid-computation)
5. Exit synchronous computation
6. Check state again → may now be aborted
7. Decide whether to continue or bail out
```

The abort **state** (inspectable at any time) and the abort **notification** (pushed
to the consumer when it happens) are complementary mechanisms. The notification enables
reactive response during async work. The state enables polling at safe checkpoints
during synchronous work.

### Principle 5: The Reason Travels With the Signal

An abort is not just "stop" — it's "stop, and here's why." The signal must carry a
**reason** (of any type, though conventionally an `Error`) that:

- Explains why the abort happened (timeout, user cancellation, parent task failure,
  resource limit exceeded)
- Can be rethrown or wrapped by the consumer
- Is available both via the notification callback and via state inspection

The reason must be set at the moment of abort and remain stable thereafter.

### Principle 6: One-Way, One-Time, Irreversible

An abort signal transitions from "not aborted" to "aborted" exactly once and never
reverts. This is a critical simplification:

- Consumers don't need to handle un-abort, re-abort, or state oscillation
- The state property is monotonic: once `true`, always `true`
- No complex state machine is required — just a boolean with a one-time transition
- Late listeners (registered after abort has already occurred) can be handled
  deterministically

### Principle 7: The Protocol Defines Receiving, Not Sending

The protocol standardizes what a **signal** looks like — the consumption interface.
How the abort is *triggered* is a separate concern. This separation matters because:

- Multiple consumers can share one signal
- The triggering mechanism can vary (explicit call, timeout, parent cancellation,
  resource pressure)
- A protocol only needs to standardize the API surface that consumers depend on
- The Web platform's `AbortController` is one possible controller; others can exist

### Principle 8: Unsubscription — "I No Longer Need to Be Told"

A consumer that registers for abort notification must be able to **withdraw that
registration** when it no longer needs to be notified. This is not merely a cleanup
convenience — it is a fundamental part of the protocol contract.

A consumer's interest in an abort signal is bounded by the lifetime of the work it is
performing. When that work completes — whether successfully, by throwing, or because
the consumer decided to stop for its own reasons — it must be able to say "I am no
longer interested in this signal." Without this:

- **Resource leaks.** A signal may be long-lived (e.g., a server-wide shutdown signal
  shared across many requests). If completed consumers cannot detach, the signal
  accumulates dead references indefinitely.
- **Composed signals cannot clean up.** A signal derived from N inputs via an `any`
  combinator subscribes to all N. When one fires, it must unsubscribe from the
  remaining N-1 (Principle 9). This is impossible without an unsubscription mechanism.
- **Unexpected side effects.** A listener that outlives its intended scope may fire
  long after the consumer has moved on, operating on stale state or closed resources.

The unsubscription mechanism must be **synchronous** and **immediate** — after
unsubscription, the callback must never be invoked again, even if abort occurs in the
same turn.

### Principle 9: Multi-Consumer by Default

A single signal is routinely shared across multiple independent consumers. A signal
passed into an API like `fetch(url, { signal })` may simultaneously be observed by:

- The fetch implementation itself
- A timeout wrapper around the fetch
- Application-level cleanup logic
- A composed signal that is monitoring it as one of several inputs

Each of these is an independent consumer that registers its own callback on the same
signal, and each must be able to subscribe and unsubscribe independently without
interfering with the others.

The protocol must therefore support **multiple concurrent consumers** on a single
signal object. A single-consumer model (like the `onabort` property pattern, where
setting a new handler silently replaces the previous one) is insufficient — it creates
a class of silent-replacement bugs where one consumer unknowingly destroys another's
registration.

`AbortSignal` is ambiguous on this point: it inherits `addEventListener` from
`EventTarget` (multi-consumer) but also exposes an `onabort` property (single-consumer,
last-writer-wins). An abort protocol should resolve this ambiguity in favor of
multi-consumer semantics as the only mode.

### Principle 10: Multiple Signals, Multiple Sources

In practice, a consumer often needs to respond to **multiple independent sources of
cancellation** at the same time. For example:

- A per-request timeout **and** a server-wide shutdown signal
- A user-initiated cancel **and** a parent task's abort signal
- A deadline from an upstream caller **and** an internal resource limit

The consumer doesn't care *which* source fires first — it needs to know "at least one
of my cancellation conditions has been met." This is the `AbortSignal.any()` pattern:
given N signals, produce a single derived signal that aborts when any input aborts.

The protocol does not need to mandate a composition primitive. However, the protocol
shape **must enable** composition to be built on top of it. Concretely, this means:

- Any object satisfying the protocol can participate in composition — the subscription
  and state-check mechanisms must be sufficient to build an `any()`-style combinator
  in userland or as a standard library utility.
- The reason must propagate: when a composed signal fires, the reason comes from
  whichever constituent signal triggered first.
- Composition is purely a consumer-side operation — it should not require access to
  the controllers that created the original signals.
- A composed signal that subscribes to N inputs must be able to unsubscribe from the
  remaining N-1 when one fires, to avoid resource leaks. This is a direct application
  of the unsubscription principle (Principle 8), and connects naturally to
  `Symbol.dispose`.

In real-world usage, only the "any" combinator (first-to-fire) has demonstrated
practical need. An "all" combinator (abort only when every signal has aborted) has no
known use case in the cancellation domain.

### Principle 11: The "Never Aborts" Degenerate Case

The protocol has a natural degenerate case for "already aborted" — a signal where the
state check returns `true` from the start and the reason is immediately available.
There is a symmetric degenerate case: a signal that **will never abort**. The state
check is `false` and will remain `false` forever. Listeners can be registered but will
never be called.

This arises constantly in practice:

- A function accepts an optional signal. When no cancellation is desired, the caller
  passes nothing. The function must then guard every signal interaction with a null
  check: `if (signal?.aborted)`, `signal?.addEventListener(...)`, etc.
- The function passes the signal down to sub-operations. Each layer repeats the null
  checks.
- A long-lived background task runs indefinitely with no cancellation source, but the
  APIs it calls still expect signals.

A "never aborts" signal eliminates this pervasive null-checking. Consumers can
unconditionally interact with the signal — check state, register listeners, pass it
down — without branching on its presence.

Critically, a never-aborts signal should **discard registrations entirely** rather than
accumulating listeners that will never be called. Since the signal can never fire, there
is no reason to hold references to callbacks. A consumer that registers and never
unsubscribes — entirely reasonable when the signal can never trigger — must not leak.
The never-aborts signal should be zero-cost: subscription is acknowledged and
immediately forgotten.

The protocol shape should be designed so that a never-aborts signal is trivially
constructible. Whether the language provides a built-in sentinel (analogous to
`AbortSignal.abort()` for the already-aborted case) is a separate question, but the
protocol must not preclude it.

### Principle 12: Late Listeners Are Ignored

If a listener is registered on a signal that has **already aborted**, the registration
is silently ignored. The listener is not called, not retained, and no error is thrown.

The protocol already provides the pre-check mechanism (Principle 3) for exactly this
purpose: consumers are expected to check `.aborted` before registering a listener. The
correct consumer pattern is:

```
1. Check .aborted → if true, handle immediately (throw, bail out, etc.)
2. Register listener
3. Begin work
```

Automatically firing late listeners would be harmful:

- It creates a hidden alternate code path that only executes under specific timing
  conditions, making bugs harder to reproduce and reason about.
- It conflates two distinct operations — state checking and event subscription — into
  a single call, violating separation of concerns.
- It encourages consumers to skip the explicit pre-check, relying on the listener
  to handle both the "already aborted" and "aborted later" cases, which leads to
  fragile code that behaves differently depending on timing.

### Principle 13: Cancellation is best-effort

- Triggering an abort signal is not guaranteed to prevent any further work from occurring.
- The consumer is not obligated to interrupt any in-progress work when the signal is triggered.
- The signal might arrive too late to actually cancel the work or the operation may not be
  interruptible.

### Principle 14: Existing solutions should fit retroactively

Specifically, it should be possible for the existing `AbortSignal` to retroactively be fitted
to a language-level abort protocol without requiring downstream code changes. `AbortSignal`
itself may need to change (e.g. by adding new properties or methods, etc) but applications
currently using `AbortSignal` should not require any changes.

## Required Capabilities

| Capability                 | Mechanism                           | When Used                    |
| -------------------------- | ----------------------------------- | ---------------------------- |
| Has abort already occurred?| Synchronous state check (boolean)   | Before starting work         |
| Notify me when abort fires | Synchronous callback registration   | During cancellable work      |
| Did I miss an abort?       | Synchronous state check (boolean)   | After uninterruptible work   |
| Why was work aborted?      | Reason value (any type)             | Anytime after abort          |
| Stop notifying me          | Unsubscription mechanism            | When work completes normally |

These five capabilities — **state check**, **synchronous notification**,
**post-hoc state check** (same mechanism as pre-check), **reason access**, and
**unsubscription** — are the minimum viable surface for an abort protocol.

## Protocol Shape

### Constraints

The protocol must be defined entirely within ECMAScript. It **cannot** normatively
reference Web platform APIs. This means the following are unavailable as building
blocks:

- **`EventTarget`** — the Web platform's generic event subscription mechanism
  (`addEventListener`, `removeEventListener`, `dispatchEvent`). This is the
  subscription model that `AbortSignal` uses. ECMAScript has no equivalent.
- **`AbortSignal`** — the concrete class the ecosystem has converged on. Defined
  in the WHATWG DOM spec, not ECMAScript.
- **`AbortController`** — the controller side. Also WHATWG DOM.
- **`Event`** — the event objects dispatched through `EventTarget`. Also WHATWG DOM.

This constraint is not merely bureaucratic. Not all JavaScript environments implement
Web APIs. A language-level protocol must work everywhere ECMAScript runs — including
embedded engines, serverless runtimes, and non-browser hosts that have no DOM.

However, the protocol must be designed so that `AbortSignal` can **retroactively
satisfy it** — ideally with no changes, or at most with the addition of a well-known
Symbol method. The Web platform should be able to layer its richer API on top of the
protocol without conflict.

## Relationship to Existing Work

### AbortController / AbortSignal (WHATWG DOM)

The Web API that the ecosystem has converged on. The protocol should be designed so
that `AbortSignal` can retroactively satisfy it. The most natural path: WHATWG adds
a `Symbol.abort` method to `AbortSignal.prototype` that returns a protocol-conforming
view (or `AbortSignal` itself, if it already conforms — which it mostly does, minus
the subscription mechanism difference).

### TC39 Proposals

- **Explicit Resource Management** (`using` / `Symbol.dispose`): Disposal and
  cancellation are related but distinct concerns. Disposing a controller cleans up
  resources (e.g., detaches listeners) but must not trigger abort (Principle 12).
  The unsubscribe handle returned by the subscription mechanism is a natural
  candidate for `Symbol.dispose` integration.
- **Observable** (Stage 1, Dormant): Observables have built-in cancellation semantics via
  subscription teardown. An abort protocol should be compatible — an Observable
  subscription could accept a signal, and the subscription's teardown could serve
  as an unsubscribe mechanism.
- **Error.cause** (ES2022): The `reason` carried by an abort signal is typically
  propagated as an error cause.
- **Error.isError** (Stage 2): Cross-realm abort signal identification may face
  similar challenges to cross-realm error identification.
- **Concurrency Control** (Stage 1): The Concurrency Control proposal would likely
  be [dependent on a abort protocol](https://github.com/tc39/proposal-concurrency-control/issues/14) mechanism.

### WHATWG Proposals

- [https://github.com/whatwg/dom/pull/1425](https://github.com/whatwg/dom/pull/1425)

## Brainstorming the design space

For those interested in just brainstorming / exploring the design space, have a look
at [BRAINSTORM_DESIGN_SPACE.md](BRAINSTORM_DESIGN_SPACE.md)
