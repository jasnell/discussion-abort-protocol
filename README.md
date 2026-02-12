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

This document defines an **abort protocol** — a minimal set of requirements that any
object can satisfy to participate in cooperative cancellation. The protocol is designed
so that `AbortSignal` already satisfies it, requiring no changes to the Web platform.

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

### Principle 12: Abort Is Always Explicit, Never Implicit

Triggering an abort must be a **deliberate, explicit act**. An abort signal must never
fire as a side effect of some other operation. In particular:

- **Disposal must not trigger abort.** If an abort controller is used with `using` /
  `Symbol.dispose`, disposing the controller means "I am done managing this controller's
  lifetime" — it does **not** mean "cancel all work associated with this signal."
  These are fundamentally different intents. A consumer that lets a controller fall out
  of scope or be disposed is relinquishing management, not issuing a cancellation order.
- **Garbage collection must not trigger abort.** A signal becoming unreachable is not
  an abort. Work may continue indefinitely without a live reference to its signal.
- **Unsubscription must not trigger abort.** A consumer removing its own listener is
  saying "I no longer care" about its own notification — not requesting that other
  consumers be interrupted.

Abort has consequences: it interrupts work, may cause side effects to be rolled back,
and propagates a reason that typically becomes an exception. An action with these
consequences must never happen as a surprise. The only way to abort is to explicitly
call the abort mechanism on the controller side.

### Principle 13: Late Listeners Are Ignored

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

### Principle 14: Cancellation is best-effort

- Triggering an abort signal is not guaranteed to prevent any further work from occurring.
- The consumer is not obligated to interrupt any in-progress work when the signal is triggered.
- The signal might arrive too late to actually cancel the work or the operation may not be
  interruptible.

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

### What the Protocol Must Express

From the principles, the protocol needs to surface exactly four things:

1. **State: has abort occurred?** — A synchronous boolean read.
2. **Reason: why?** — A synchronous read of an arbitrary value.
3. **Subscribe: notify me when abort occurs.** — Register a callback.
4. **Unsubscribe: stop notifying me.** — Remove a previously registered callback.

Capabilities (1) and (2) are straightforward properties. The interesting design
question is (3) and (4): the subscription mechanism.

### The Subscription Problem

ECMAScript has no general-purpose event subscription primitive. The patterns that
exist in the ecosystem are:

**`EventTarget` (Web API — not available)**
```js
signal.addEventListener('abort', fn);
signal.removeEventListener('abort', fn);
```
Multi-consumer, but requires function reference identity for removal, and depends
on a Web API.

**`.on` / `.off` (Node.js `EventEmitter` — not standardized)**
```js
signal.on('abort', fn);
signal.off('abort', fn);
```
Same shape as `EventTarget` but with different method names. Also not ECMAScript.

**Single handler property (DOM `on*` pattern)**
```js
signal.onabort = fn;
signal.onabort = null;
```
Single-consumer only. Violates Principle 9 (multi-consumer).

**Subscribe-returns-unsubscribe (Observable / reactive pattern)**
```js
const unsubscribe = signal.subscribe(fn);
// later:
unsubscribe();
```
Multi-consumer. Each subscription returns its own independent teardown function.
No need for function reference identity — the returned handle is the unsubscription
token. This pattern exists in ECMAScript today (no Web API dependency) and is used
by numerous libraries (RxJS, MobX, Redux, Solid, Svelte, etc.).

### Current WHATWG Proposal

There is a current [WHATWG](https://github.com/whatwg/dom/pull/1425) proposal
that has been recently opened that suggests adding an `addAbortCallback(...)`
API to `AbortSignal` that would exist in addition to `addEventListener('abort', ...)`

### Subscribe-Returns-Unsubscribe

The subscribe-returns-unsubscribe pattern has significant advantages for this protocol:

- **Independent unsubscription.** Each call to subscribe returns a unique handle.
  Consumers don't need to hold a reference to their original callback to unsubscribe.
  This is especially important when the callback is an anonymous arrow function,
  which is the overwhelmingly common case.
- **No reference identity requirement.** `removeEventListener` requires passing the
  exact same function reference. This fails silently when the consumer wraps or
  rebinds the callback — a common source of bugs in `EventTarget`-based code.
- **Naturally multi-consumer.** Each subscription is independent. No single-handler
  slot to fight over.
- **Composable with `using`.** The returned unsubscribe function can be made
  disposable (`Symbol.dispose`), allowing `using unsub = signal.subscribe(fn)` to
  automatically clean up when the scope exits.
- **No dependency on Web APIs.** The entire mechanism is plain ECMAScript — a method
  call returning a function.

### Observables as a Subscription Mechanism?

The dormant TC39 Observable proposal (Stage 1, now largely replaced by the WICG
effort) would provide a standardized subscription model with built-in teardown semantics.
In principle, a signal could expose an Observable that emits once (the abort reason)
and then completes, with subscription teardown serving as the unsubscribe mechanism:

```js
const subscription = signal.aborted$.subscribe({
  next(reason) { /* abort occurred */ }
});
// later:
subscription.unsubscribe();
```

However, Observable is likely too heavyweight for this use case:

- **Observables model multi-value async streams.** An abort signal is a single,
  one-time event. The Observable machinery — next/error/complete callbacks,
  completion semantics, error channels — is unnecessary overhead for a boolean
  state transition.
- **Conceptual mismatch.** Observables are pull-to-subscribe, push-to-deliver,
  with rich lifecycle. An abort signal is a simple notification with no sequence,
  no completion distinct from the event itself, and no error channel (the reason
  *is* the value, not an error in the Observable sense).
- **Dependency ordering.** If the abort protocol depends on Observable, it cannot
  ship until Observable does. Observable has been at Stage 1 for a considerable
  time. The abort protocol should be self-contained.

More fundamentally, Observable itself is likely to be a **consumer** of the abort
protocol. An Observable subscription may need to accept an abort signal to allow
external cancellation of the stream. If the abort protocol were defined in terms of
Observable, this would create a circular dependency — Observable needs the abort
protocol, but the abort protocol needs Observable. The abort protocol must be a
lower-level primitive that Observable (and other higher-level abstractions) can
build upon, not the other way around.

The subscribe-returns-unsubscribe pattern described above is *compatible* with
Observable — an Observable adapter could be layered on top without the protocol
itself depending on it.

The other obvious key issue is that the Observable proposal in TC-39 is effectively
dead and has been replaced by the WICG proposal, which cannot be normatively
referenced in TC-39 (it's a Web platform API so has same dependency constraints
as `AbortSignal`).

### Protocol Identification

How does a consumer know an object satisfies the abort protocol? Three options:

**Option A: Well-known Symbol (like `Symbol.iterator`)**

A new well-known symbol (e.g., `Symbol.abort`) that, when called, returns a protocol
object. This is the most TC39-idiomatic approach — it's exactly how iterables,
disposables, and other protocols are defined.

```js
const signal = obj[Symbol.abort]();
signal.aborted  // boolean
signal.reason   // any
signal.subscribe(fn) // → unsubscribe function
```

**Option B: Duck typing (like thenables)**

Any object with the right shape is treated as a signal. No symbol needed.

```js
// Anything with .aborted, .reason, and .subscribe is a signal
```

Simpler adoption but weaker identification. Risks false positives from objects
that coincidentally have these property names.

**Option C: Concrete class**

A new built-in `CancelSignal` class (or similar). The most complete approach but
the largest surface area, and harder to retrofit onto `AbortSignal`.

### Open Design Questions

1. **Protocol identification mechanism.** Symbol-based, duck typing, or concrete
   class? A Symbol is the most consistent with TC39 precedent for protocols.

2. **Shape of the protocol object.** Is the subscribe method directly on the signal,
   or does a Symbol method return a separate subscription-capability object (paralleling
   how `Symbol.iterator` returns a separate iterator object)?

3. **Should the unsubscribe return be disposable?** If the unsubscribe function has
   a `Symbol.dispose` method (or *is* the `Symbol.dispose` method), it integrates
   with `using` for automatic cleanup. This is attractive but adds a dependency on
   the Explicit Resource Management proposal.

## Design Examples

The following examples illustrate how each protocol option could look in practice
across the key use cases: basic consumption, the `AbortSignal` compatibility bridge,
the never-aborts sentinel, and signal composition.

Note: These are **NOT** intended to be concrete proposals. These are meant not to
propose a solution but to give a starting point to discuss and contrast the design
principles. Take each with a grain of salt. I tend to think best in terms of concrete
examples to help ground my thoughts, so this is all just pure brainstorming.

### Option A: Well-Known Symbol Protocol

A new `Symbol.abort` is introduced. An object is "cancelable" if it has a
`[Symbol.abort]()` method that returns a protocol object with `aborted`,
`reason`, and `subscribe`.

#### Basic Consumer

```js
async function doWork(signal) {
  // Obtain the protocol object
  const cancel = signal[Symbol.abort]();

  // Pre-check (Principle 3)
  if (cancel.aborted) throw cancel.reason;

  // Subscribe (Principle 1)
  const unsubscribe = cancel.subscribe((reason) => {
    // Synchronous notification (Principle 2)
    cleanup();
  });

  try {
    const result = await someAsyncWork();

    // Post-check (Principle 4)
    if (cancel.aborted) throw cancel.reason;

    return result;
  } finally {
    // Unsubscribe (Principle 8)
    unsubscribe();
  }
}
```

#### AbortSignal Compatibility

`AbortSignal` already has `.aborted` and `.reason`. However, because the protocol
object returned by `[Symbol.abort]()` is a separate object, these must be
proxied through as getters. The subscription must also be bridged from
`EventTarget` to the subscribe-returns-unsubscribe pattern.

WHATWG would add one method to `AbortSignal.prototype`:

```js
AbortSignal.prototype[Symbol.abort] = function () {
  const signal = this;
  return {
    get aborted() { return signal.aborted; },
    get reason() { return signal.reason; },
    subscribe(fn) {
      signal.addEventListener('abort', fn, { once: true });
      return () => signal.removeEventListener('abort', fn);
    }
  };
};
```

Note that each call to `[Symbol.abort]()` allocates a new wrapper object.
This is consistent with how `Symbol.iterator` works (each call returns a fresh
iterator), but for a protocol object that is typically obtained once and held for
the duration of the work, the allocation cost is minimal.

#### Never-Aborts Sentinel

```js
const neverAborts = {
  [Symbol.abort]() {
    return {
      aborted: false,
      reason: undefined,
      subscribe(_fn) { return () => {}; }  // discard, no-op unsubscribe
    };
  }
};

// Consumer code works uniformly:
await doWork(neverAborts);
```

#### Composition (any)

```js
function any(signals) {
  // Obtain protocol objects for all inputs
  const cancels = signals.map(s => s[Symbol.abort]());

  // Check if any input is already aborted (Principle 3)
  for (const cancel of cancels) {
    if (cancel.aborted) {
      return alreadyAborted(cancel.reason);
    }
  }

  // Build a composite signal
  let subscribers = [];
  let aborted = false;
  let reason;
  const unsubscribes = [];

  const composite = {
    get aborted() { return aborted; },
    get reason() { return reason; },
    subscribe(fn) {
      if (aborted) return () => {};  // already fired, ignore (Principle 13)
      subscribers.push(fn);
      return () => {
        subscribers = subscribers.filter(f => f !== fn);
      };
    },
    [Symbol.abort]() { return this; }
  };

  // Subscribe to all inputs
  for (const cancel of cancels) {
    const unsub = cancel.subscribe((r) => {
      if (aborted) return;  // another input already fired
      aborted = true;
      reason = r;

      // Unsubscribe from all inputs (Principle 8)
      for (const u of unsubscribes) u();

      // Notify composite subscribers synchronously (Principle 2)
      for (const fn of subscribers) fn(r);
      subscribers = [];
    });
    unsubscribes.push(unsub);
  }

  return composite;
}
```

#### Observations on Option A

- **Two-step access.** Consumers call `[Symbol.abort]()` before interacting
  with the signal. This parallels `[Symbol.iterator]()`. Unlike iterators, where
  the distinction between "iterable" and "iterator" is meaningful (an iterable can
  produce multiple independent iterators), a cancel signal has singular state —
  there is no analogous reason to separate the source from the protocol object.
- **Protocol object identity.** The source object can return itself (as arrays do
  for `Symbol.iterator`). If so, the protocol surface is directly on the signal
  object, and the Symbol method serves as identification. This collapses the
  distinction between Options A and D.
- **Symbol-based identification.** A well-known Symbol cannot collide with existing
  string-named properties. The Symbol method produces the protocol object, so
  calling it inherently yields an object with the expected shape.

### Option B: Duck Typing

No Symbol. Any object with `aborted` (boolean), `reason` (any), and `subscribe`
(function returning unsubscribe function) is treated as a signal.

#### Basic Consumer

```js
async function doWork(signal) {
  // Pre-check
  if (signal.aborted) throw signal.reason;

  // Subscribe
  const unsubscribe = signal.subscribe((reason) => {
    cleanup();
  });

  try {
    const result = await someAsyncWork();

    // Post-check
    if (signal.aborted) throw signal.reason;

    return result;
  } finally {
    unsubscribe();
  }
}
```

#### AbortSignal Compatibility

`AbortSignal` already has `.aborted` and `.reason`, which satisfy two of the three
protocol requirements out of the box. Only the subscription mechanism needs bridging.

WHATWG would add one method to `AbortSignal.prototype`:

```js
AbortSignal.prototype.subscribe = function (fn) {
  this.addEventListener('abort', fn, { once: true });
  return () => this.removeEventListener('abort', fn);
};
```

This is the lightest retrofit of any option — one method addition, no wrapper objects,
no Symbols. `AbortSignal` satisfies the protocol directly.

#### Never-Aborts Sentinel

```js
const neverAborts = Object.freeze({
  aborted: false,
  reason: undefined,
  subscribe(_fn) { return () => {}; }
});
```

#### Observations on Option B

- **Direct access.** No `[Symbol.abort]()` call. Consumers interact directly
  with the signal's properties and methods.
- **`AbortSignal` retrofit.** One method addition (`.subscribe`). The existing
  `.aborted` and `.reason` properties already conform. No wrapper objects.
- **No formal identification.** An object with an unrelated `.aborted` property and
  a `.subscribe` method would satisfy the protocol shape. The collision risk depends
  on the specificity of the property names chosen.
- **Name collision.** The name `subscribe` is used by other patterns (Observable,
  pub/sub libraries). A more specific name would reduce collisions but may conflict
  with existing properties (e.g., `onabort` on `AbortSignal` is single-handler).

### Option C: Concrete Built-in Class

A new `CancelSignal` class is added to ECMAScript, with a corresponding
`CancelController` for the trigger side.

#### Basic Consumer

```js
async function doWork(signal) {
  // Pre-check
  if (signal.aborted) throw signal.reason;

  // Subscribe
  const unsubscribe = signal.subscribe((reason) => {
    cleanup();
  });

  try {
    const result = await someAsyncWork();

    // Post-check
    if (signal.aborted) throw signal.reason;

    return result;
  } finally {
    unsubscribe();
  }
}
```

#### Controller Side

```js
const controller = new CancelController();
const signal = controller.signal;

// Pass signal to consumers
doWork(signal);

// Later, trigger abort
controller.cancel(new Error('timeout'));
```

#### AbortSignal Compatibility

This is the hardest option to retrofit. `AbortSignal` is an existing class with its
own prototype chain, and it cannot retroactively extend or mix in a new built-in
class. There are several possible paths, none fully satisfying:

**Path 1: Spec-level coordination.** WHATWG redefines `AbortSignal` to extend
`CancelSignal` (or implement a shared interface). This requires cross-spec
coordination between TC39 and WHATWG, and would be a breaking change if
`AbortSignal`'s prototype chain is altered.

```js
// AbortSignal would inherit .subscribe from CancelSignal.prototype
// But AbortSignal already has its own .aborted and .reason — potential conflicts
```

**Path 2: Protocol Symbol.** `AbortSignal` implements the protocol via a well-known
Symbol, essentially falling back to Option A or D for interop:

```js
AbortSignal.prototype[Symbol.abort] = function () {
  return this;
};
AbortSignal.prototype.subscribe = function (fn) {
  this.addEventListener('abort', fn, { once: true });
  return () => this.removeEventListener('abort', fn);
};
```

But if a Symbol is needed anyway, the concrete class adds little over Option D.

**Path 3: Bridge utility.** A static method wraps an `AbortSignal`:

```js
const cancelSignal = CancelSignal.from(abortSignal);
```

This works but means every API that accepts a `CancelSignal` must also accept an
`AbortSignal` and wrap it, or callers must remember to wrap. This is a persistent
source of friction.

#### Never-Aborts Sentinel

```js
// Could be a static property
const signal = CancelSignal.none;

// Or a static factory
const signal = CancelSignal.never();
```

#### Observations on Option C

- **API surface.** A concrete class can provide static methods (`CancelSignal.any()`,
  `CancelSignal.never()`, `CancelSignal.from()`) and `instanceof` checking.
- **Specification footprint.** Defines both a signal class and a controller class,
  with constructors, prototypes, and internal slots.
- **`AbortSignal` retrofit.** `AbortSignal` cannot retroactively extend
  `CancelSignal`. The relationship requires duck typing, a shared protocol Symbol,
  or spec-level coordination between TC39 and WHATWG.
- **Coexistence with `AbortSignal`.** In environments that have both, there are two
  signal types serving the same purpose. APIs must decide which to accept, or accept
  both and bridge between them.
- **`instanceof` and realms.** `instanceof` provides identification within a single
  realm but does not work across realm boundaries.

### Option D: Symbol Protocol With Direct Conformance

A hybrid of Options A and B. A `Symbol.abort` is used for identification, but
the protocol properties (`aborted`, `reason`, `subscribe`) live directly on the
object — the Symbol method simply returns `this`.

#### Basic Consumer

```js
async function doWork(signal) {
  // Pre-check
  if (signal.aborted) throw signal.reason;

  // Subscribe
  const unsubscribe = signal.subscribe((reason) => {
    cleanup();
  });

  try {
    const result = await someAsyncWork();
    if (signal.aborted) throw signal.reason;
    return result;
  } finally {
    unsubscribe();
  }
}
```

In practice, consumers never call `[Symbol.abort]()` — they use the properties
directly. The Symbol serves only as a **type brand** for identification:

```js
// A library that accepts an optional signal:
function startWork(options) {
  const signal = options?.signal;
  if (signal && !signal[Symbol.abort]) {
    throw new TypeError('signal does not satisfy the cancelation protocol');
  }
  // ... use signal.aborted, signal.subscribe, etc. directly
}
```

#### AbortSignal Compatibility

`AbortSignal` already has `.aborted` and `.reason`. WHATWG would add two things:
the Symbol brand and the `.subscribe` method:

```js
AbortSignal.prototype[Symbol.abort] = function () { return this; };
AbortSignal.prototype.subscribe = function (fn) {
  this.addEventListener('abort', fn, { once: true });
  return () => this.removeEventListener('abort', fn);
};
```

This is nearly as light as Option B (one extra line for the Symbol brand), with the
added benefit of reliable identification. Since `[Symbol.abort]()` returns
`this`, there is no wrapper allocation — `AbortSignal` instances *are* the protocol
objects.

#### Never-Aborts Sentinel

```js
const neverAborts = Object.freeze({
  aborted: false,
  reason: undefined,
  subscribe(_fn) { return () => {}; },
  [Symbol.abort]() { return this; }
});
```

#### Observations on Option D

- **Direct access with Symbol identification.** Consumers interact with the signal's
  properties directly. The Symbol is present for identification but not required for
  access.
- **The Symbol is a brand, not a factory.** Unlike `Symbol.iterator` (which returns
  a new iterator each time), `[Symbol.abort]()` always returns `this`. A
  cancel signal has singular state — there is no analogous reason to produce
  independent protocol objects from the same source.
- **`AbortSignal` retrofit.** Adds two things: the Symbol and `.subscribe`.
  Existing `.aborted` and `.reason` properties already conform. No wrapper objects.
- **Disconnected identification.** The Symbol asserts protocol conformance, but the
  consumer accesses `.aborted`, `.reason`, and `.subscribe` as named properties on
  the same object. The Symbol's presence does not guarantee the existence or
  correctness of those properties. A buggy implementation could have
  `[Symbol.abort]` but missing or incorrect properties. Consumer code that
  wants to be robust must still verify the properties exist, regardless of whether
  the Symbol is present.

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
