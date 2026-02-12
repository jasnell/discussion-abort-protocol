---

**NOTE: Much of the following is just design brainstorming around the problem
space... it may resemble concrete proposals (and in one case, there is a link
to a concrete proposal over in WHATWG) but it's mostly brainstorming to help
ground the discussion a bit. - Signed, A Firm Believer In Cunningham's Law**

---

## What the Protocol Must Express

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

There is a current WHATWG proposal ([https://github.com/whatwg/dom/pull/1425](https://github.com/whatwg/dom/pull/1425))
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
