---
theme: default
title: "ECMAScript Abort Protocol"
info: |
  Discussion: An abort protocol for ECMAScript

  Seeding a design discussion for cooperative cancellation
  at the language level.
class: text-center
drawings:
  persist: false
transition: slide-left
mdc: true
---

# ECMAScript Abort Protocol

A discussion on cooperative cancellation at the language level

<!--
This is a discussion item, not a concrete proposal. The goal is to explore
the design space and identify principles for an abort protocol that could
be standardized in ECMAScript.
-->

---

# Objective

Before opening a proposal, can we get alignment on:

- **The problem** — is there a gap that needs filling?
- **The principles** — what requirements must a solution satisfy?
- **The scope** — what should (and shouldn't) be standardized?

<br>

This is a **discussion**, not a proposal. No spec text or stage ask today.

---

# The Gap

**Cancellation is fundamental** to async and long-running work.

The ecosystem has converged on `AbortController` / `AbortSignal`.

But these are **WHATWG DOM APIs**, not ECMAScript.

- TC39 cannot normatively reference Web APIs
- Not all JS environments implement them
- The language has no built-in concept of cancellation
- Yet the entire ecosystem depends on one

---

# Proposed Problem Statement

<br><br>

**The abort signaling pattern the ecosystem depends on should be defined in ECMAScript.**

<br><br>

(Designed so that `AbortSignal` can **retroactively satisfy it** with
minimal or no changes to the Web platform.)

---
layout: section
---

# Design Principles

---

# Principle 1: Receiving a Signal

The core requirement: a way for a **consumer** to be told "stop what you're doing."

<br>

One-way communication:

**Controller** (decides to cancel) → **Consumer** (does cancellable work)

<br>

The consumer needs a way to register interest in being notified when abort occurs.

---

# Principle 2: Synchronous & Immediate

Notification **must** be synchronous — same turn, no microtask/macrotask deferral.

- Consumer may be about to commit a side effect — async deferral means
  the side effect happens anyway
- Cancellation is time-sensitive — abort must fire *now*, not after the
  next promise settles
- Matches existing `AbortSignal` behavior

<br>

> An abort signal is an **interrupt**, not a suggestion.

---

# Principle 3 & 4: State Checks

### Pre-check: "Has abort already occurred?"

Before starting work, check if the signal is already aborted.

```js
if (signal.aborted) throw signal.reason;
```

### Post-check: "Did abort occur while I was busy?"

After uninterruptible synchronous work, check again.

```
1. Check state      → not aborted
2. Register listener → begin work
3. Synchronous computation (can't respond mid-work)
4. Check state again → may now be aborted
```

Same mechanism (boolean state), two distinct use cases.

---

# Principle 5: Reason Travels With the Signal

An abort is not just "stop" — it's **"stop, and here's why."**

<br>

The signal carries a **reason** (any type, conventionally an `Error`):

- Why: timeout, user cancellation, parent task failure, resource limit
- Can be rethrown or wrapped by the consumer
- Available via both notification callback and state inspection
- Set at the moment of abort, stable thereafter

---

# Principle 6: One-Way, One-Time, Irreversible

A signal transitions from "not aborted" → "aborted" **exactly once**.

<br>

- No un-abort, re-abort, or state oscillation
- Monotonic: once `true`, always `true`
- No complex state machine — just a boolean with a one-time transition

---

# Principle 7: Receiving, Not Sending

The protocol defines **what a signal looks like** — the consumption interface.

How abort is *triggered* is a separate concern.

- Multiple consumers can share one signal
- Trigger mechanism varies (explicit call, timeout, parent cancellation)
- Protocol only standardizes what consumers depend on
- `AbortController` is one possible controller; others can exist

---

# Principle 8: Unsubscription

A consumer must be able to say **"I no longer need to be told."**

- Consumer interest is bounded by the lifetime of the work
- Without unsubscription:
  - Long-lived signals leak dead references
  - Composed signals cannot clean up after one input fires
  - Stale listeners fire on closed resources
- Unsubscription must be **synchronous and immediate**
  - After unsubscription, callback must never be invoked again

---

# Principle 9: Multi-Consumer

A single signal is routinely shared across multiple independent consumers.

```
fetch(url, { signal })    ← fetch implementation
  ↑                        ← timeout wrapper
  ↑                        ← application cleanup logic
  ↑                        ← composed signal monitoring
```

- Each consumer subscribes and unsubscribes independently
- Single-consumer models (`onabort` property) create silent-replacement bugs
- `AbortSignal` is ambiguous: `addEventListener` (multi) vs `onabort` (single)

---

# Principle 10: Multiple Sources

Consumers often need **multiple cancellation sources** simultaneously.

- Per-request timeout **and** server shutdown signal
- User cancel **and** parent task abort
- Upstream deadline **and** internal resource limit

<br>

The protocol does not mandate a composition primitive, but its shape
**must enable** building `any()`-style combinators.

Only the "any" combinator (first-to-fire) has demonstrated practical need.

---

# Principle 11: The "Never Aborts" Signal

A signal that **will never abort** — the symmetric degenerate case.

- Eliminates pervasive null-checking when no cancellation is desired
- Consumers interact uniformly — no `if (signal?.aborted)` everywhere
- **Discards registrations entirely** — zero-cost, no leak path
- Protocol shape should make this trivially constructible

---

# Principle 12: Late Listeners Are Ignored

Registering a listener on an **already-aborted** signal: silently ignored.

```
1. Check .aborted → if true, handle immediately
2. Register listener
3. Begin work
```

- Consumers check state first (Principle 3), then register
- Auto-firing late listeners creates hidden timing-dependent code paths
- Conflates state checking and event subscription

---

# Principle 13: Cancellation is Best-Effort

Triggering an abort signal is **not guaranteed** to prevent further work.

- The consumer is not obligated to interrupt in-progress work
- The signal might arrive too late to actually cancel the operation
- The operation may not be interruptible

---

# Principle 14: Existing Solutions Should Fit

`AbortSignal` should **retroactively fit** the protocol without breaking downstream code.

- `AbortSignal` itself may need changes (new properties or methods)
- Applications currently using `AbortSignal` should require **no changes**
- The protocol must be designed with this constraint in mind

---
layout: center
class: text-center
---

# Discussion

Do these principles capture the requirements for an abort protocol?

What's missing? What's contentious?
