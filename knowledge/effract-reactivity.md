# Effract State Management & Reactivity — Architecture Report

## 1. Does effract use effect-ts Atom?

**Partially.** It imports `AtomRef` from `effect/unstable/reactivity` (`packages/effract/src/infrastructure/reactivity-core.ts:16`) as the **backing storage cell** for its atoms, but builds its **own full subscription + dependency tracking system** on top. It does **not** use `@effect/state` or any other effect-ts atom library.

The custom atom interfaces are defined in `packages/effract/src/domain/protocol.ts:267-291`:

- `ReadableAtom<A>` — `.value`, `.subscribe()`, `.derive()`, `[Symbol.iterator]`
- `Atom<A>` — extends `ReadableAtom` with `.set()`, `.update()`

The implementation in `packages/effract/src/infrastructure/reactivity-core.ts:146-181` wraps an `AtomRef` for storage, adds a `Set<() => void>` listener set per atom, and fires notifications on write — with change-detection gating via `Effect/Equal.equals` (no-op writes skip notification entirely).

---

## 2. How Reactivity Works

### Storage Layer

Every atom is backed by `AtomRef.make(initial)` from `effect/unstable/reactivity`. This is a thread-safe cell reachable from any Effect program — no React dependency at the storage level. This is what lets services hold atoms without needing a client context.

### Subscription Layer (Custom)

Each atom has its own `Set<() => void>` listener set. `subscribe()` adds a callback, returns an unsubscribe function and the atom. On `set()`/`update()`, `notify(listeners)` fires all registered callbacks.

Key reference: `reactivity-core.ts:64-75`:
```
if (batchDepth > 0) {
  // defer into pending batch
} else {
  // snapshot + fire inline
}
```

### Change Detection

All writes gate on `Equal.equals(next, ref.value)` (`reactivity-core.ts:152-154`). If the new value is equal to the current value (structurally, property-order-insensitive), the write is a no-op — no notification, no re-render downstream. This uses Effect v4's structural `Hash`/`Equal`.

### Batching (`batch()`)

Module-level `batchDepth` counter + `pending` set (`reactivity-core.ts:50-126`). Writes inside `batch()` defer notifications to the `pending` set; on outermost flush, each collected listener fires exactly once. Synchronous-only — an async callback throws in dev mode to prevent depth corruption across interleaved requests on the server.

### Derived Atoms / Computation

The `Computation` class (`reactivity-core.ts:437-534`) implements the core tracking engine for both `derive()` and the `observe` hook:

1. **Evaluate** — Run the selector with a tracing `read()` function that records every atom read.
2. **Track** — On first subscriber, subscribe to each discovered dependency.
3. **React** — On dep notification, re-evaluate, re-subscribe if the dependency *set* changed (reference identity), and notify own listeners if value changed.
4. **Lazy unsubscribe** — When the last subscriber unsubscribes, all dep subscriptions are dropped.

Dependency comparison is **reference identity** of the atom objects (`dep === currentDeps[i]`), checked in `sameDeps()` (`reactivity-core.ts:454-455`).

#### Async Derived (`derive.effect`)

`reactivity-core.ts:608-650` — A two-step yieldable that first yields a tracked `deriveReadonly` (subscribing the component), then yields a `query` keyed by the snapshot of source values. When a source atom changes, the component re-renders, the new snapshot differs from the query key, and the effect refetches.

### React Binding

`packages/effract/src/infrastructure/react/reactivity.tsx` — The `'use client'` half of reactivity:

| Hook | Mechanism |
|---|---|
| `useAtomValue(atom)` | `useSyncExternalStore(atom.subscribe, getValue, getValue)` |
| `useAtomSet(atom)` | `useCallback` returning a setter/updater |
| `useAtom(atom)` | Tuple of `useAtomValue` + `useAtomSet` |
| `observe(($) => ...)` | Creates a `Computation` in `useRef`, drives it with `useSyncExternalStore` |
| `<Observe>{($) => ...}</Observe>` | Render-prop form of `observe` |

The interpreter's `reader` port (`reactivity.tsx:91-97`) is how `yield* atom` works in a REC — it calls `useSyncExternalStore` for each yielded atom, in order.

---

## 3. The In-Render Interpreter (Fiber Bridge)

`packages/effract/src/application/interpreter.ts:116-155` — The heart of the system. `driveRec` runs **inside React's render pass** and walks the generator synchronously:

```
function driveRec(gen, deps, state) {
  let step = gen.next();
  while (!step.done) {
    const instruction = step.value;
    if (isHook)           → state.hooks += 1;  unwrap inline value
    if (isPlacement)      → deps.placer.place() → createElement for child REC
    if (isSuspensable)    → state.hooks += 2;  resolver.resolve() → useSyncExternalStore + use()
    if (isAtom)           → state.hooks += 1;  reader.read() → useSyncExternalStore
    else (Effect)         → resolveEffect() → runSyncExit or suspend
    step = gen.next(result);
  }
  return step.value;
}
```

### How Suspension Works (`resolveEffect`, `interpreter.ts:71-108`)

1. Call `deps.executor.runSyncExit(effect)` — runs the Effect synchronously if possible.
2. If synchronous → return value immediately.
3. If the effect is async, `runSyncExit` returns an `Exit.failure` whose squashed cause is an `AsyncFiberError` **carrying the still-running fiber**.
4. This fiber is handed to the suspensable resolver, which:
   - Stores it in the query cache (one execution, not two)
   - Calls `useSyncExternalStore` (subscribe to cache entry version)
   - Calls `React.use()` on the entry's promise → suspends
5. On retry (after promise settles), the cache hit means the promise resolves instantly.

The `state.hooks` counter is the invariant that makes `.catch` safe: every hook-bearing yield is counted *before* it may throw or suspend, so a caught failure that skipped later hooks re-throws to an error boundary instead of rendering a fallback inline (which would violate the Rules of Hooks). Pinned by test `#37`.

---

## 4. Lifecycle Scopes (Outside React's Rendering)

The system defines **four distinct lifecycle scopes**:

| Scope | Mechanism | File Reference |
|---|---|---|
| **Module-global** | Atoms declared outside `make()` at module level. Process-wide, no isolation — documented footgun. | `reactivity-core.ts` module-level state |
| **Per-request (server)** | Fresh `ManagedRuntime` built per render, disposed after response. Default behavior; `{ scope: 'shared' }` opts out. | `server/mount.ts:113` |
| **Runtime (per `mount`)** | `ManagedRuntime` built in `<Runtime>` provider. Keyed queries dedup by `runtimeScopeOf(executor)` — stable per runtime identity via `WeakMap`. | `suspensable-store.ts:117-124` |
| **Component-instance** | Instance ID minted via `nextInstanceId()`, kept in `useRef`. Claims on cache entries reconciled on commit. | `suspensable-store.ts:126-128`, `rec.tsx:102-108` |

### Query / Suspensable Lifecycle

`packages/effract/src/infrastructure/react/suspensable-store.ts` — The runtime-level query cache (556 lines).

**Key data structures:**

```
store      Map<entryId, Entry>        — all entries
buckets    Map<bucketHash, Set<Entry>> — dedup groups (keyed or positional)
slots      Map<slotKey, Set<Entry>>   — position groups (for supersede)
claims     Map<instanceId, Set<entryId>> — per-instance claim bookkeeping
```

**Entry fields** (`Entry` interface, `suspensable-store.ts:46-83`): `id`, `slot`, `bucket`, `runtimeScope`, `key`, `tags`, `staleTime`, `refetch`, `promise`, `cancel`, `status`, `value`, `fetchedAt`, `revalidating`, `version`, `listeners`, `claims`, `sweep`.

**Lifecycle steps:**

1. **Creation** (`createEntry`, line 286) — Cache miss. For query effects, `attachRunner` (line 351) provides the `refetch` routine using `executor.runPromise` with `AbortController`.
2. **Resolution** (`makeSuspensableResolver`, line 218) — Each render reads two hooks: `useSyncExternalStore` (subscribe to entry version) + `suspender.use(promise)` (React's `use`). On cache hit, redundant fiber is interrupted immediately.
3. **Stale-while-revalidate** (line 267) — If `staleTime > 0` and entry is past freshness, serve stale value + kick off background refetch.
4. **Claims reconciliation** (`reconcileClaims`, line 405) — On committed render, increment claims for used entries, decrement for unused. Zero-claim entries are evicted (interrupted + swept after 10s).
5. **Full teardown** (`releaseClaims`, line 424) — On unmount, all instance claims released.

### Component-Scoped Resources (`scoped()`)

`domain/protocol.ts:239-247` — Runs the effect against a fresh `Scope` that is a child of the runtime's scope. The store keeps `close()` as the entry's canceller. On unmount, `entry.cancel()` closes the scope deterministically — finalizers run exactly once, with no race against runtime disposal.

### Runtime Disposal

`packages/effract/src/infrastructure/react/runtime.tsx` — The `<Runtime>` provider renders an internal error boundary. A render that builds a runtime but is discarded before commit (sibling throws) is caught by this boundary, which disposes the runtime only if it was never adopted. A committed runtime is owned by a StrictMode-safe unmount latch (`useEffect` cleanup). This avoids the `setTimeout` race and `FinalizationRegistry` limitations documented in ADR 0002.

### Server Mount

`packages/effract/src/application/server-driver.ts` — The server path resolves everything up-front: collects all yielded effects, runs them via `ManagedRuntime.runPromise`, waits for all suspensables, then renders the full tree. No `'use client'` hooks are available — it's purely async with no in-render interpreter.

---

## 5. Cache Keys and Dedup

| Yield Kind | Cache Identity | Scope |
|---|---|---|
| `suspend(effect)` | `(runtimeScope, componentScope, index)` — positional | Per component instance position |
| `query(effect, key)` | `(runtimeScope, Hash.hash(key))` + `Equal.equals(key)` confirmation | Runtime-wide |
| `query.enabled(effect, key, enabled)` | Same as query, but `key` is `[key, enabled]` | Runtime-wide |
| `yield* asyncEffect` (raw) | Same as `suspend` — positional, load-once | Per component instance position |
| `scoped(effect)` | Same as `suspend` — positional, load-once | Per component instance position |

Keys are compared with Effect v4's structural `Hash`/`Equal` — property-order-insensitive for plain objects (`{a:1,b:2}` equals `{b:2,a:1}`), identity-based for opaque class instances (`suspensable-store.ts:204-211`).

---

## 6. ADR-to-Code Mapping

| ADR | Topic | Key Files |
|---|---|---|
| 0001 | Fiber reconciliation — the in-render interpreter | `domain/protocol.ts`, `application/interpreter.ts`, `application/ports.ts` |
| 0002 | Async lifecycle hardening — fiber joining, per-request isolation, runtime disposal | `suspensable-store.ts`, `server/mount.ts`, `runtime.tsx` |
| 0003 | Data story — query cache, mutations, streams | `suspensable-store.ts`, `domain/protocol.ts` (`query`, `suspend`, `scoped`) |
| 0004 | Universal state — serialization, hydration, scoped | `reactivity-core.ts` (`serializable`, `dehydrate`, `hydrate`), `hydration.tsx`, `suspensable-store.ts` (`seedQuery`), `server-driver.ts` |
| 0005 | DX moat — devtools, testing, `.catch` retry | `devtools-core.ts`, `react/devtools.tsx`, `react/catch-boundary.tsx` |

---

## 7. Summary of Key Architectural Patterns

| Concern | Implementation | File:Line |
|---|---|---|
| **Storage** | Effect's `AtomRef` (thread-safe cell) | `reactivity-core.ts:147` |
| **Change detection** | `Equal.equals` — structural equality, no-op writes skip notify | `reactivity-core.ts:152` |
| **Subscription** | `Set<() => void>` per atom; `subscribe()` returns unsubscribe | `reactivity-core.ts:170-175` |
| **React binding** | `useSyncExternalStore` per atom read | `react/reactivity.tsx:48-55` |
| **Dependency tracking** | `Computation` — collect deps by running selector with tracing `read()` | `reactivity-core.ts:437-534` |
| **Batching** | Module-level `batchDepth` + `pending` set; outermost flush | `reactivity-core.ts:50-126` |
| **Generator driving** | `driveRec` walks generator synchronously inside render, dispatches by type | `application/interpreter.ts:116-155` |
| **Suspense** | All async effects through React's `use()` via suspensable store | `suspensable-store.ts:218-281` |
| **Query dedup** | Buckets by `(runtimeScope, Hash.hash(key))`; matched by `Equal.equals(key)` | `suspensable-store.ts:204-211` |
| **Server isolation** | Fresh runtime per render (default); `scope: 'shared'` to opt out | `server/mount.ts` |
| **Component lifecycle** | Instance ID in `useRef`; claims reconciled on commit, released on unmount | `suspensable-store.ts:405-433` |
| **Runtime disposal** | Error boundary in provider catches discarded renders; committed owned by unmount latch | `runtime.tsx` |



## React thinks in priorities

React has ideas like

It can delay rendering.

It can interrupt rendering.

It can restart rendering.

It can merge rendering.

Effract appears to have

```
atom changed

↓

run immediately
```

No scheduler.

No lanes.

No priorities.

---

## Imagine typing

Every keystroke

```
keypress

↓

atom.set()

↓

Equal.equals()

↓

notify

↓

derived

↓

derived

↓

React
```

Now imagine simultaneously

* websocket updates
* notifications
* cache invalidations
* background polling

All of those look identical to Effract.

React would prefer

```
typing first

background later
```

Effract appears unable to distinguish them.

---

# Could priorities be added?

I think yes.

Actually, I think they're one of the most interesting directions for a library like this.

Instead of

```ts
atom.set(value)
```

imagine

```ts
atom.set(value, {
    priority: "urgent"
})
```

or

```ts
atom.set(value, {
    lane: Input
})
```

Then notifications don't fire immediately.

Instead

```
enqueue update

↓

scheduler

↓

flush highest priority first
```

Now you're much closer to React's architecture.

---

## Even better: split invalidation from recomputation

Right now it sounds like

```
set

↓

recompute

↓

notify
```

What if it became

```
set

↓

mark dirty

↓

scheduler

↓

recompute later
```

That's actually closer to how Solid, Vue, and React internally amortize work.

Instead of eagerly recomputing everything on every write, you accumulate invalidations and process them according to policy.

---

## `Equal.equals` could become lazy

Right now

```
write

↓

Equal.equals

↓

notify
```

But another possibility is

```
write

↓

mark dirty

↓

later...

↓

compute

↓

Equal.equals
```

Now ten writes become

```
write

write

write

write

↓

one equality check
```

instead of four.


Typing into an input doesn't usually benefit from a structural comparison—it changes on almost every keystroke. Doing a deep equality check there is effectively paying for a comparison whose answer is almost always "different."

Conversely, configuration objects or query keys may benefit from structural equality because they're updated infrequently and avoiding unnecessary downstream work is valuable.

---

## Where I think Effract could evolve

If I were evolving this architecture, I'd probably introduce **three distinct phases**:

```
Write
```

Very cheap.

Just store the new value.

Mark the atom dirty.

---

```
Schedule
```

Assign a priority.

Merge duplicate updates.

Coalesce writes.

Possibly debounce background work.

---

```
Compute
```

Run `Equal.equals`.

Recompute derived values.

Notify React.

That separation gives you a lot more flexibility.

---

## One subtle point

There's one caveat: React itself has strict rules around `useSyncExternalStore`. When an external store changes, React expects the snapshot to reflect the latest committed state. If you delay the actual mutation of the store to implement priorities, you can violate those expectations and introduce tearing or stale reads.

A safer design is to separate:

* **mutation** (store the latest value immediately),
* **invalidation** (mark dependents dirty immediately), and
* **recomputation/notification** (schedule according to priority where semantics allow).

That preserves external store correctness while still letting you amortize expensive derived computations.



## Effect is actually a very good foundation for Scheduler

Since Effract already depends on Effect, I'd lean into its runtime rather than introducing another scheduler.

Effect already has many of the primitives you'd need:

* `Queue` for incoming writes
* `PubSub` for notifications
* `Schedule` for throttling/debouncing/retrying
* `Fiber` for lightweight concurrent tasks
* `Semaphore` / `Mailbox` for coordination
* `Clock` for time-based batching
* `Ref` / `SynchronizedRef` for mutable state
* cooperative cancellation and interruption

What's missing is the policy layer that decides *when* to propagate reactive updates.

---

## Imagine an AtomScheduler

Instead of

```ts
atom.set(value)
```

you might have

```ts
scheduler.enqueue(atom, value)
```

The scheduler owns the propagation pipeline.

Internally it might look something like:

```
Queue<AtomWrite>

↓

coalesce by atom

↓

priority queue

↓

batch

↓

Equal.equals

↓

notify
```

The important point is that **writes are just events**. They don't immediately become state transitions.



## Effect's Schedule is underused here

One thing I rarely see in frontend state libraries is using Effect's scheduling combinators for propagation.

Imagine something like:

```ts
atom.schedule(
    Schedule.fixed("16 millis")
)
```



Now propagation is a first-class policy instead of being hardcoded as "immediate."

---

## React's priorities could also be integrated

This is where I think things become genuinely novel.

Suppose writes carried priorities:

```ts
atom.set(value, {
    priority: "input"
})
```

or

```ts
priority: "background"
```

Your scheduler could maintain separate queues:

```
Urgent

Normal

Background

Idle
```

When React begins a render, only the urgent queue is flushed.

Background updates can wait.

This starts to align with React's lane model without needing to reimplement React's scheduler.



## I actually think there's an even bigger opportunity

Reading through Effract's architecture, I noticed that `Computation` is doing **both** dependency tracking and scheduling.

Those are separate concerns.

A cleaner architecture might look like:

```
          Atom
           │
           ▼
    Dependency Graph
           │
           ▼
 Invalidator (mark dirty)
           │
           ▼
 Reactive Scheduler
           │
           ▼
    Recompute Graph
           │
           ▼
 Structural Equality
           │
           ▼
      React Notify
```

Now each stage has one responsibility:

* **Dependency graph** knows *who depends on whom*.
* **Invalidator** marks affected nodes as dirty.
* **Scheduler** decides *when* work should happen, batches updates, and assigns priorities.
* **Recompute** recalculates dirty derived values.
* **Equality** suppresses notifications when values are effectively unchanged.
* **React bridge** delivers the final notifications via `useSyncExternalStore`.

That separation opens the door to batching, debouncing, animation-frame flushing, idle processing, and priority queues without complicating the dependency graph itself.

In fact, this starts to resemble how modern incremental computation systems (such as spreadsheet engines, build systems like Bazel/Buck, and self-adjusting computation research) are structured: **change detection, scheduling, and recomputation are distinct phases** rather than one synchronous operation. I think that's the direction that would give Effract capabilities that go beyond existing React state libraries rather than just being another fine-grained reactive system.
