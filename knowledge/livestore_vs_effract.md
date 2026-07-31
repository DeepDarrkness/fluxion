# Livestore vs Effract — Architecture Comparison

> Two approaches to building reactive state management with Effect-TS.
>
> **Livestore** (`livestore.dev`) — Event-sourced state management using Effect-TS as a background runtime.
> State is stored in SQLite, mutated via events, and propagated through a custom reactive dependency graph.
> React bindings are imperative callbacks glued onto the graph.
>
> **Effract** — Generator-based React components using Effect-TS as a render-time programming model.
> Atoms are backed by `AtomRef` from `effect/unstable/reactivity`, computed via a tracing derivation engine,
> and driven inside React's render pass by an in-render generator interpreter.
>
> This document compares the architectural decisions across nine dimensions, explains the trade-offs,
> and includes annotated code snippets showing the same concept implemented in both systems.

---

## Table of Contents

1. [Effect-TS Integration Philosophy](#1-effect-ts-integration-philosophy)
2. [Reactivity Core](#2-reactivity-core)
3. [React Bridge](#3-react-bridge)
4. [Suspense & Async](#4-suspense--async)
5. [State Mutation Model](#5-state-mutation-model)
6. [Dependency Tracking](#6-dependency-tracking)
7. [Query & Caching](#7-query--caching)
8. [Lifecycle & Scoping](#8-lifecycle--scoping)
   - [Atomicity, Writers, and Performance](#85-atomicity-writers-and-performance)
9. [Key Takeaways](#9-key-takeaways)
10. [Appendix — Same Concept, Two Systems](#10-appendix--same-concept-two-systems)

---

## 1. Effect-TS Integration Philosophy

### Livestore

Effect-TS is the **orchestration runtime** — it manages concurrency, resource safety, and streaming
for the sync engine, but is almost entirely **invisible at the React layer**.

```
┌────────────────────────────────────────────────────────┐
│  React Components                                      │
│  • imperative hook calls: useQuery, store.commit()     │
│  • no Effect code, no generators, no yield*            │
└──────────────┬─────────────────────────────────────────┘
               │ calls throw/catch style
               ▼
┌────────────────────────────────────────────────────────┐
│  Store (class)                                         │
│  • runs Effect programs internally via runSyncWith()   │
│  • exposes commit/subscribe/query as sync methods      │
└──────────────┬─────────────────────────────────────────┘
               │ Effect-managed background
               ▼
┌────────────────────────────────────────────────────────┐
│  Sync Engine (Effect runtime)                          │
│  • FiberHandle, TxQueue, Stream, ManagedRuntime        │
│  • background pull stream + push batching              │
│  • all invisible to the component layer                │
└────────────────────────────────────────────────────────┘
```

Key places Effect is used:
- `createStore` returns an `Effect` that sets up `Scope`, adapter, sync processor
- `ClientSessionSyncProcessor` uses `FiberHandle`, `TxQueue`, `Queue`, `Stream` for sync orchestration
- `store.commit()` internally calls `Effect.runSyncWith(services)` — the commit function is an imperative wrapper around an Effect program
- OTel spans are managed via Effect's tracing

The **user never writes `yield*`** in their React code. The Store class exposes a classic imperative API.

### Effract

Effect-TS is a **render-time programming model** — users write Effect-style generators (RECs: Reactive Effect
Components) that run inside React's render pass.

```
┌────────────────────────────────────────────────────────┐
│  React Components (REC generators)                     │
│  • function* MyComp(): ReactElement => {               │
│  •   const count = yield* counterAtom                  │
│  •   const data = yield* query(db.getAll, key)         │
│  •   return <div>{count} {data}</div>                  │
│  • }                                                   │
└──────────────┬─────────────────────────────────────────┘
               │ driveRec walks generator on every render
               ▼
┌────────────────────────────────────────────────────────┐
│  Interpreter (driveRec)                                │
│  • runs inside render pass                             │
│  • dispatches by yield type:                           │
│    - yield* atom → useSyncExternalStore                │
│    - yield* effect → runSyncExit or React.use()        │
│    - yield* placement → createElement                  │
│  • maintains state.hooks counter for Rules of Hooks    │
└──────────────┬─────────────────────────────────────────┘
               │ synchronously reads / suspends
               ▼
┌────────────────────────────────────────────────────────┐
│  AtomRef & Runtime                                     │
│  • atoms backed by effect/unstable/reactivity          │
│  • query cache with ManagedRuntime                     │
│  • suspense boundaries via React.use()                 │
└────────────────────────────────────────────────────────┘
```

Key places Effect is used:
- Every `yield*` in a component goes through the interpreter
- `runSyncExit(effect)` — tries synchronous execution, falls back to suspense
- `ManagedRuntime` per request (server) or per `<Runtime>` provider (client)
- Atoms use `AtomRef` from `effect/unstable/reactivity` as backing storage

### Comparison

| Dimension | Livestore | Effract |
|-----------|-----------|---------|
| Effect visibility | Hidden behind imperative Store API | Directly exposed in component generators |
| User writes `yield*`? | Never | Every render |
| Async execution | Background fibers, sync commit API | In-render via `React.use()` / suspense |
| Effect scope | Per-store `ManagedRuntime` | Per-request / per-`<Runtime>` provider |
| Error handling | Effect caught internally, exposed as sync errors | Effect Exit handled by interpreter, caught by error boundaries |

### Why This Matters

Livestore's approach is more **familiar** to developers accustomed to Redux/MobX/etc — the Store class
looks like a normal state management library. The complexity of Effect-TS is walled off in the infrastructure
layer. However, this also means users cannot leverage Effect's compositional power in their components
(no `Effect.gen`, no `Stream`, no structured concurrency at the component level).

Effract's approach is more **expressive** — effects, async, and atoms compose naturally via generators.
But it requires deeper understanding of Effect-TS and the in-render interpreter, and introduces new
abstractions (RECs, `driveRec`, `state.hooks`) that are unfamiliar to most React developers.

---

## 2. Reactivity Core

### Livestore — `ReactiveGraph`

A custom dependency graph with three node types:

| Node | Role | Mutable? | Dirty tracking |
|------|------|----------|----------------|
| `Ref<T>` | Mutable cell. Holds a value, can be `set` externally. | Yes | `isDirty` always `false` — refs are never dirty, only their downstream is |
| `Thunk<T>` | Pure computation. Re-evaluates when dependencies change. | No (derived) | `isDirty` flag, set by `markSuperCompDirtyRec` |
| `Effect` | Side-effect. Runs callback when dependencies change. | N/A | Collected in set, always runs on refresh |

Edges are **bidirectional**:
- `node.sub` = atoms this node depends on (upstream)
- `node.super` = thunks/effects that depend on this node (downstream)

**Refresh propagation:**
```
setRef(tableRef, val)
  → markSuperCompDirtyRec(ref, effectsSet)
    → for each superComp:
        thunk → thunk.isDirty = true, recurse
        effect → add to effectsSet
  → runEffects(effectsSet)
    → for each effect: doEffect()
      → effect calls get(dependency)
        → compute(dependency):
            dirty → recompute, track new deps, compare equality
            clean → return cached result
```

**Equal: Livestore — `reactive.ts`**
```typescript
const thunk = {
  previousResult: NOT_REFRESHED_YET,
  isDirty: true,
  computeResult: () => {
    // Reset sub computations
    thunk.sub = new Set()
    // Run with tracing getAtom
    const result = getResult(getAtom, setDebugInfo, ctx, otelContext, debugRefreshReason)
    const resultChanged = !thunk.equal(thunk.previousResult as T, result)
    thunk.isDirty = false
    thunk.previousResult = result
    return result
  },
  equal: options?.equal ?? deepEqual,
  sub: new Set(),    // upstream deps
  super: new Set(),  // downstream dependents
}
```

### Effract — `AtomRef` + `Computation`

A simpler model built on Effect's `AtomRef`:

| Concept | Role | Mutable? |
|---------|------|----------|
| `AtomRef<T>` | Thread-safe cell (from `effect/unstable/reactivity`) | Yes |
| `atom(initial)` | Wraps `AtomRef` + `Set<() => void>` listeners | Yes |
| `ReadableAtom<T>` | `.value`, `.subscribe()`, `.derive()` | No (read-only) |
| `Computation` | Derivation engine — tracks deps via tracing `read()` | No (derived) |

**Refresh propagation:**
```
atom.set(newValue)
  → Equal.equals(next, current)? no-op if equal
  → update ref value
  → notify(listeners):
      if batchDepth > 0: defer to pending set
      else: fire each listener immediately
```

**Equal: Effract — `reactivity-core.ts`**
```typescript
private set(value: A): void {
  const ref = this.ref as unknown as { value: A }
  if (Equal.equals(value, ref.value)) return  // no-op gating
  ref.value = value
  this.notify(this.listeners)
}
```

### Comparison

| Dimension | Livestore ReactiveGraph | Effract AtomRef + Computation |
|-----------|-------------------------|-------------------------------|
| Backing store | Plain object fields | `AtomRef` from `effect/unstable/reactivity` |
| Node types | 3 (Ref, Thunk, Effect) | 2 (Atom, Computation) |
| Edge direction | Bidirectional (sub + super) | Unidirectional (Composition knows deps via `read()`) |
| Refresh strategy | Eager — mark dirty, run effects | Push — notify listeners on write |
| Equality | Custom per thunk, defaults to `deepEqual` | Global `Equal.equals` (structural) |
| Cut-off | Per-thunk — skips recomputation if deps unchanged, always runs effects | Per-atom — no-op write skips all downstream |
| Batching | Deferred effects + `skipRefresh` | Module-level `batchDepth` counter |

### Why This Matters

Livestore's `ReactiveGraph` is more sophisticated but also more complex — three node types, bidirectional edges,
eager dirty marking. This complexity is motivated by the need to track **table-level granularity**: a SQL
query thunk depends on specific table refs, and when any row in that table changes, the thunk must
re-execute. The thunk's `results$` node does the actual DB query, so the cache lives at the graph level.

Effract's model is simpler — atoms notify listeners directly. Derivations recompute by re-running the
selector with a tracing `read()` that collects dependencies. This works well when dependencies are
other atoms (fine-grained, stable references). It would need additional machinery to support
database-backed invalidation.

---

## 3. React Bridge

### Livestore

React is bridged to the reactive graph via **`Effect` nodes** that call `setState` under `React.useEffect`:

```
// useQuery (simplified)
const initialResult = query$.run(...)             // sync initial read
const [valueRef, setValue] = useStateRef(...)      // store in ref

React.useEffect(() => {
  return store.subscribe(query$, (newValue) => {
    if (deepEqual(newValue, valueRef.current) === false) {
      setValue(newValue)                           // trigger React re-render
    }
  })
}, [...])
```

The `store.subscribe` call creates a reactive graph `Effect`:

```typescript
const effect = reactivityGraph.makeEffect((get) => {
  const result = get(query$.results$)
  if (!suppressCallback) onUpdate(result)    // → setValue → re-render
})
```

Key characteristics:
- No `useSyncExternalStore` — state held in `useRef`, re-render forced via counter increment
- `ReactDOM.unstable_batchedUpdates` set as `effectsWrapper` on the graph context
- `useStateRefWithReactiveInput` synchronizes ref value without extra render cycles
- Store references are reference-counted via `useRcResource`

### Effract

React is bridged via **`useSyncExternalStore`** for atomic reads, and the **`driveRec` interpreter**
for generator-based RECs:

```
// useAtomValue (simplified)
const useAtomValue = (atom) => {
  const subscribe = React.useCallback((cb) => atom.subscribe(cb), [atom])
  const getSnapshot = React.useCallback(() => atom.value, [atom])
  return React.useSyncExternalStore(subscribe, getSnapshot, getSnapshot)
}
```

For the REC generator path:

```
// driveRec (simplified, running inside render)
function driveRec(gen, deps, state) {
  let step = gen.next()
  while (!step.done) {
    const instruction = step.value
    if (isAtom)    → state.hooks += 1;  reader.read() → useSyncExternalStore
    if (isEffect)  → resolveEffect() → runSyncExit or React.use()
    if (isHook)    → state.hooks += 1;  unwrap inline value
    step = gen.next(result)
  }
  return step.value
}
```

Key characteristics:
- Standard `useSyncExternalStore` for each atom read
- `state.hooks` counter ensures Rules of Hooks safety when generators throw/catch
- `driveRec` runs the entire generator synchronously each render, dispatching by yield type
- The interpreter is what makes `yield* atom` work in a component

### Comparison

| Dimension | Livestore | Effract |
|-----------|-----------|---------|
| Subscription primitive | `Effect` node in ReactivityGraph → `setState` | `useSyncExternalStore(atom.subscribe)` |
| Initial value | Sync via `query$.run()` | Sync via `getSnapshot` |
| Re-render trigger | `setValue(newValue)` from effect callback | React's internal `forceStoreRerender(fiber)` |
| Batching | `ReactDOM.unstable_batchedUpdates` on effects wrapper | `batch()` with module-level depth counter |
| Hook rules safety | N/A — no generators | `state.hooks` counter in interpreter |
| Component API | `useQuery(def)`, `useStore(options)` | `useAtom(atom)`, `observe($ => ...)`, `function* REC` |

### Why This Matters

Livestore's bridge is designed for **imperative consumption** — the reactive graph pushes updates to
callbacks, which React picks up via standard `useEffect`. This is simpler to understand but means
React re-renders are always a downstream effect of graph propagation, never part of the propagation
itself.

Effract's bridge is designed for **declarative consumption** — the generator describes exactly what
atoms and effects the component needs, and the interpreter makes it work within React's rendering
model. This enables co-location of data dependencies with rendering logic but requires understanding
the interpreter contract.

Notably, Livestore **does not use `useSyncExternalStore`** — the standard React API for external
state. Instead it uses a pattern of `useRef` + forced re-render. Effract uses `useSyncExternalStore`
canonically, which gives better integration with React's concurrent features (automatic batching,
transition handling).

---

## 4. Suspense & Async

### Livestore

**No suspense integration for data fetching.** The Store's API is synchronous from the user's perspective:

- `store.commit(events)` — synchronous, runs Effect internally via `runSyncWith`
- `store.query(def)` — synchronous, returns current value or runs query
- `store.subscribe(def, cb)` — returns unsubscribe, cb fires synchronously during propagation

The only Suspense boundary is in `useStore`, which uses `React.use()` for the initial store loading:

```typescript
const storeOrPromise = storeRegistry.getOrLoadPromise(options)
const store = storeOrPromise instanceof Promise
  ? React.use(storeOrPromise)    // Suspends while store loads
  : storeOrPromise
```

Background async (sync with backend) happens on Effect fibers invisible to React. When sync events
arrive, they materialize and trigger the reactive graph, which calls `setState` — React sees this as
a normal state update.

### Effract

**Deep suspense integration** — all async effects go through `React.use()` via the suspensable store:

```
// In the interpreter, for async effects:
if (synchronous) → return value immediately
else             → suspensableResolver.resolve(effectFiber)
                  → useSyncExternalStore(subsribeToEntry)
                  → React.use(entry.promise)  // suspends
                  → on retry, cache hit resolves instantly
```

Three yield patterns involve suspense:

| Pattern | Cache Key | Re-fetch |
|---------|-----------|----------|
| `suspend(effect)` | Positional (component, index) | On re-mount |
| `query(effect, key)` | `(runtimeScope, Hash.hash(key))` | On key change or revalidation |
| `query.enabled(effect, key, enabled)` | Same + enabled flag | Conditional |

Server rendering resolves everything up-front — runs all effects via `ManagedRuntime.runPromise`,
waits for all suspensables, then renders.

### Comparison

| Dimension | Livestore | Effract |
|-----------|-----------|---------|
| Suspense integration | Only for store loading (`React.use()`) | All async effects (`React.use()`) |
| Data fetching pattern | Sync query + background sync | `suspend()` / `query()` with suspense |
| Server rendering | N/A (client-local state) | Eager resolve, full tree render |
| Async caching | No cache (state is DB) | Keyed query cache, stale-while-revalidate |
| Background refresh | Sync pull stream → materialize → re-render | `staleTime` + background refetch |

### Why This Matters

Livestore avoids suspense because its data model doesn't need it — the SQLite database is a synchronous
data source. Queries are always available (no "loading" state for local queries). Async is limited to
sync with remote backends, which happens transparently.

Effract embraces suspense because effects are inherently async — API calls, file reads, etc. The
query cache provides deduplication, debouncing, and stale-while-revalidate semantics that Livestore
doesn't need (the database handles all that).

The trade-off: Livestore has no loading spinners for data queries (it's always in the DB), but
requires a sync engine to propagate remote changes. Effract must handle loading states but has a
simpler path to remote data.

---

## 5. State Mutation Model

### Livestore — Event Sourcing

All state changes are **immutable events**:

```
store.commit(event)                        // User commits event
  → syncProcessor.encodeEvents(event)      // Assign sequence number
  → syncProcessor.materializeEvents(...)   // Run materializer → SQL writes
  → syncProcessor.push(...)                // Merge into sync state, queue for leader
  → reactivityGraph.setRefs(tableRefs)     // Trigger reactive propagation
```

Events are:
- Immutable — once committed they cannot be changed
- Sequenced — composite sequence numbers (client seq + rebase generation)
- Stored — eventlog database for replay and sync
- Materialized — pure functions project events into state

Materializers are pure functions that return SQL operations:

```typescript
const todoCreated = ({ id, text }) => todos.insert({ id, text, completed: false })
const todoUpdated = ({ id, text, completed }) =>
  todos.update({ ...omitUndefined({ text, completed }) }).where({ id })
```

### Effract — Direct Mutation (Current Model)

State is just **mutable atoms**:

```
atom.set(newValue)          // Direct replacement
atom.update(prev => ...)    // Functional update
```

No event log, no materializers, no sequencing. Atoms are plain state cells.

This describes the current Effract atom API, not the proposed event-runtime
architecture. The proposed frontend runtime would add a Foldkit-style ordered
Event -> State transition layer while retaining Effract atoms and Computations
as the fine-grained projection mechanism.

### Comparison

| Dimension | Livestore | Effract |
|-----------|-----------|---------|
| Mutation primitive | `store.commit(event)` | `atom.set(value)` / `atom.update(fn)` |
| State transformation | Materializers (pure fn → SQL) | Direct assignment |
| History | Full event log, replayable | None (unless manually implemented) |
| Sync/offline | Built-in: sync protocol, rebase, rollback | Not built-in (app-level) |
| Time travel debugging | Possible (replay events) | Not without event log |
| Optimistic updates | Supported via local commit + push/pull | App-level |
| Migration | Schema versioning, migration hooks | N/A |

For the proposed Effract architecture, the relevant distinction becomes:

```text
Frontend Event runtime
  -> ordered pure transition
  -> canonical state AtomRef
  -> Effract Computation projections
  -> React
```

Direct atom mutation remains appropriate for ephemeral local UI state, but
canonical feature/domain state should normally enter through the event runtime.

### Why This Matters

Livestore's event sourcing is the defining architectural decision. It enables:
- Rebase-based conflict resolution (SQLite changesets for rollback)
- Full audit trail
- Sync protocol where events flow between clients
- Deterministic replay for testing

But it comes with significant complexity — the sync state machine, sequence numbers, materializers,
changeset tracking. For apps that don't need multi-device sync or audit trails, this is overkill.

Effract's model is simpler and more familiar — `atom.set(value)` is the mental model of React's
`useState`. It pairs naturally with server-rendered apps where data comes from API calls rather
than a local database.

---

## 6. Dependency Tracking

### Livestore — Automatic Bidirectional Edges

Dependencies are tracked **automatically** during thunk execution. When a thunk calls `get(atom)`,
the graph calls `addEdge(thunk, atom)`:

```typescript
const getAtom = (atom) => {
  this.addEdge(thunk, atom)    // thunk.sub.add(atom); atom.super.add(thunk)
  return compute(atom, ...)
}
```

Edges are **re-established on every recomputation** — the thunk resets `sub = new Set()` before
re-executing, so stale dependencies are automatically dropped.

This means:
- No manual dependency arrays
- Dependencies change dynamically based on control flow
- The graph is always consistent with the current computation

### Effract — Tracing via `Computation.read()`

Dependencies are tracked by running the selector with a **tracing proxy**. The `Computation` class:

```typescript
// During Computation evaluation:
const read = (atom) => {
  deps.add(atom)
  return atom.value
}
const result = selector(read)
// deps now contains all atoms that were read
```

On dependency notification, the Computation:
1. Re-evaluates the selector
2. Compares new deps to old deps by **reference identity** (`dep === currentDeps[i]`)
3. If deps changed, unsubscribes from removed deps, subscribes to new ones
4. If value changed, notifies own listeners

```typescript
private sameDeps(newDeps: unknown[], oldDeps: unknown[]): boolean {
  if (newDeps.length !== oldDeps.length) return false
  return newDeps.every((dep, i) => dep === oldDeps[i])
}
```

### Comparison

| Dimension | Livestore | Effract |
|-----------|-----------|---------|
| Tracking method | Automatic via `get()` during thunk execution | Tracing via `read()` proxy during selector eval |
| Edge comparison | N/A — edges re-created each run | Reference identity of atom objects |
| Dynamic deps | Supported — control flow changes deps automatically | Supported — `read()` is called inside selector |
| Cleanup | Old edges removed by resetting `sub = new Set()` | Unsubscribe from removed atoms |
| Overhead | Edge objects per dependency | Set membership + reference comparison |
| Table-level deps | Built-in — SQL parser extracts table names | N/A |

### Why This Matters

Both approaches achieve the same result — derived value that updates when dependencies change.
The key difference is **what constitutes a "dependency"** :

Livestore tracks **fine-grained data sources** (table refs) as dependencies. A DB query thunk depends
on `tableRef_todos`, not on individual atom objects. This is the right level for database-backed
state — you say "this query depends on the todos table", not "this query depends on these specific
atoms".

Effract tracks **atom references** as dependencies. A derived atom depends on other atoms by identity.
This is the right level for in-memory state — you say "this value depends on `counterAtom` and
`filterAtom`", not "this value depends on an abstract data source".

---

## 7. Query & Caching

### Livestore

Queries are **reactive graph nodes** — they re-execute when their table ref dependencies change:

- `queryDb(def)` — creates a `LiveStoreDbQuery` with a `results$` thunk that runs DB queries
- `computed(fn)` — creates a `LiveStoreComputedQuery` with a `results$` thunk that runs the selector
- `signal(init)` — wraps a `Ref` for local state

There is **no cross-component query cache** — each unique SQL query executes against the database
whenever its table refs change. `RcRef` (reference counting) ensures multiple components using the
same `LiveQueryDef` share a single `LiveQuery` instance, but the query still re-executes on every
invalidation.

The `SqliteDbWrapper` has a **result cache** (`QueryCache`) that avoids re-executing identical SQL
queries with identical bind values, but this is a per-store cache invalidated by table writes:

```typescript
class QueryCache {
  #entries: Map<CacheKey, any>      // (sql + bindValues) → result
  #dependencies: Map<TableName, Set<CacheKey>>  // table → dependent cache keys
  invalidate(tables: Set<string>)  // removes all cached results touching these tables
}
```

### Effract

Queries use a **runtime-scoped cache** with dedup and lifecycle management:

- `query(effect, key)` — runs effect, caches result by `(runtimeScope, Hash.hash(key))`
- `suspend(effect)` — runs effect, caches by position (component + index), no dedup

The suspensable store has:

```typescript
store      Map<entryId, Entry>            — all entries
buckets    Map<bucketHash, Set<Entry>>    — dedup groups (keyed query dedup)
slots      Map<slotKey, Set<Entry>>       — position groups (for supersede)
claims     Map<instanceId, Set<entryId>>  — per-instance claims
```

Lifecycle:
1. **Create entry** — cache miss, runs effect, stores promise + cancel
2. **Hit** — existing entry, no re-execution
3. **Stale-while-revalidate** — serve stale + background refetch
4. **Claims** — per-instance claims reconciled on commit; zero-claim entries evicted
5. **Teardown** — `entry.cancel()` closes the scoped effect's `Scope`

### Comparison

| Dimension | Livestore | Effract |
|-----------|-----------|---------|
| Cache scope | Per-store (SQL result cache) | Per-runtime (effect result cache) |
| Dedup | N/A — DB handles dedup | `(runtimeScope, key)` buckets |
| Staleness | Always fresh (DB is source of truth) | `staleTime` + revalidation |
| Eviction | Table-based invalidation | Claims-based: zero claims → 10s → evict |
| Query identity | Query string + bind values | `Hash.hash(key)` + `Equal.equals(key)` |
| Cross-component sharing | RcRef shares LiveQuery instance | Cache dedup shares entry |

### Why This Matters

Livestore doesn't need a sophisticated cache because the **database is the cache**. SQLite is
in-process and fast enough that re-querying on invalidation isn't a bottleneck. The result cache
(`QueryCache`) is an optimization, not a caching layer.

Effract needs a real cache because effects are async and potentially expensive (API calls, etc.).
The query cache provides keyed dedup, stale-while-revalidate, and proper lifecycle management —
features that are essential for network data but unnecessary for local DB queries.

---

## 8. Lifecycle & Scoping

### Livestore — Two-Tier

**Store level:**
- `RcMap` in `defRcMap` — reference-counted `LiveQuery` instances per reactivity graph
- `withRCMap(hash, makeFn)` — creates or reuses query instances by hash
- Destroyed when last `RcRef.deref()` reaches 0

**Component level:**
- `useRcResource(scope, key, create, dispose)` — component-level RC lifecycle
- `WeakMap<scope, Map<key, Entry>>` — per-store cache of RC resources
- Entry destroyed when RC reaches 0 (component unmount or key change)

```
Components ──useRcResource──▶ RC'd query instances
Store       ──defRcMap──▶   RC'd LiveQuery/Signal instances
Adapter     ──Scope──▶      SQLite connections, fibers
```

### Effract — Four-Tier

| Tier | Scope | Mechanism | Cleanup |
|------|-------|-----------|---------|
| Module | Process-wide | Module-level atoms | Never (documented footgun) |
| Request (server) | Per render | Fresh `ManagedRuntime` | Disposed after response |
| Runtime (client) | Per `<Runtime>` | `ManagedRuntime` built in provider | Error boundary + unmount latch |
| Component | Per instance | Instance ID in `useRef` | Claims reconciliation |

```
Module atoms ──── global (no cleanup)
Request runtime ── scoped to one SSR render (dispose on response)
Runtime provider ─ scoped to mount (dispose on unmount)
Component ──────── instance ID + claims (release on unmount)
```

### Comparison

| Dimension | Livestore | Effract |
|-----------|-----------|---------|
| Scope tiers | 2 (store, component) | 4 (module, request, runtime, component) |
| RC mechanism | RcRef + useRcResource | Claims reconciliation |
| Isolation | Per-store | Per-runtime (keyed queries scoped) |
| Server isolation | N/A | Fresh runtime per render |
| Memory leak protection | RC reaches 0 → destroy | Zero claims → timer → evict |

### Why This Matters

Livestore has simpler scoping because there's less to manage — the database owns the actual state,
so query instances are just wrappers around reactive graph nodes. Destroying a query instance is
cheap because the data lives in SQLite.

Effract has more tiers because it must manage Effect `Scope` lifetimes, query cache entries, and
runtime resources. The four-tier system ensures proper isolation — a server render doesn't leak
into other renders, a component unmount doesn't keep cache entries alive forever, etc.

---

## 8.5 Atomicity, Writers, and Performance

This section describes the actual Livestore implementation and the implications
for the proposed Effract frontend event runtime.

### 8.5.1 What SQLite enforces

In `Store.commit`, Livestore:

1. Encodes events and assigns client sequence numbers.
2. Materializes the events into the local SQLite database.
3. Pushes the encoded events to the synchronization leader.
4. Refreshes affected table references after materialization.

For multiple events, `Store.commit` calls `SqliteDbWrapper.txn(materialize)`.
The materializer resolves each event definition, produces SQL statements, runs
them against SQLite, tracks affected tables, and can capture SQLite changesets.

Therefore SQLite provides local materialization atomicity:

```text
Event A SQL
Event B SQL
Event C SQL
  -> one SQLite transaction
  -> all commit or all roll back
```

Queries do not refresh table references until materialization has completed. The
database prevents observers of the database from seeing a partially committed
SQL batch.

SQLite does not by itself provide the complete Livestore event protocol. It does
not determine the ordering of events produced by separate clients, match a
pending event against an upstream event, or decide when a local optimistic
event must be rebased.

### 8.5.2 Does SQLite provide multiple writers?

SQLite can safely handle multiple callers attempting to write to one database,
but it serializes physical write access. It provides database locking and
transaction isolation, not parallel mutation of the same database pages.

Livestore's normal multi-client model is different:

```text
Client A -> local SQLite A
Client B -> local SQLite B
Client C -> local SQLite C
        \       |       /
          sync leader
```

Each client can optimistically materialize events in its own local database. The
sync protocol then coordinates the event chains. The leader/upstream sequence
and `SyncState` determine how pending events relate to accepted upstream events.

The division of responsibility is:

```text
SQLite       -> local transaction atomicity
Event IDs    -> local event ordering and identity
SyncState    -> pending/upstream heads and merge decisions
Changesets   -> rollback for rebase
Leader       -> authoritative upstream sequence
```

### 8.5.3 How Livestore handles concurrent local histories

Each session maintains:

```ts
type SyncState = {
  pending: EncodedEvent[]
  upstreamHead: CompositeSequence
  localHead: CompositeSequence
}
```

On local push, the client appends pending events to its local chain. On an
upstream advance, it applies new upstream events and retains pending local
events. On divergence, it:

1. Interrupts the leader push fiber.
2. Clears the leader push queue.
3. Rolls back pending events using captured SQLite changesets.
4. Applies upstream events.
5. Reassigns sequence numbers to pending events.
6. Requeues the rebased events.
7. Restarts the push fiber.

The client sync processor accumulates affected tables and refreshes table refs
after the materialization sequence. This prevents reactive queries from seeing
each intermediate rollback and replay operation.

Local multi-writer behavior is therefore not implemented by SQLite alone. It is
SQLite plus sequence validation, pending-event tracking, changesets, and the
rebase algorithm.

### 8.5.4 Effract's single writer

Effract should use one writer for canonical state per frontend application
runtime:

```text
React events / transport / timers / Effects
                    |
                    v
              Event queue
                    |
                    v
          one transition processor
                    |
                    v
          canonical state + projections
```

This is not a global process writer and it does not serialize all frontend
work. It serializes only short canonical state transitions. HTTP requests,
WebSocket reads, Effect fibers, React rendering, Suspense, and pure component
state remain independently scheduled.

The single writer is usually not a significant bottleneck because JavaScript
already executes synchronous state mutation on one thread. Without a writer,
two synchronous writes cannot truly run in parallel; they can only interleave
through callbacks and risk lost updates, reentrant reducers, or partial
snapshots.

The performance cost is the queue and transition work itself:

- event enqueue/dequeue;
- immutable state allocation;
- reducer/materializer execution;
- projection invalidation;
- notification bookkeeping.

For ordinary UI transitions this should be small. The actual bottleneck is an
expensive reducer, projection, or large immutable copy, not the fact that one
processor owns ordering.

### 8.5.5 Fast input versus slow backend work

Slow backend work does not block the writer:

```text
LoadRequested
  -> state = loading
  -> fork HTTP Effect
  -> writer is immediately available

HTTP completion
  -> TodosLoaded event
  -> short state transition
```

Rapid input can create queue pressure if every keystroke becomes a canonical
application event. The runtime should support coalescing, debouncing, and
priority policies. For example, keep a form draft in local React state and
dispatch only `SearchSubmitted` or a debounced `SearchChanged` event.

```text
pure React draft state
  -> debounce/submit
  -> canonical event
  -> command
```

Events that must be processed should retain FIFO ordering. Events that merely
represent the latest value can be coalesced.

### 8.5.6 Pure React state and multiple browser sessions

Effract should preserve pure React component state for local interaction:

- form drafts;
- hover/focus state;
- animation state;
- temporary input state;
- component-only interaction details.

Use the central event runtime for shared, replayable, server-related, or
workflow state. This prevents the event queue from becoming a bottleneck for
every keystroke.

One mounted application has one root Scope, event queue, canonical state, and
transition processor. Separate application mounts should normally have separate
runtimes. Separate tabs have separate frontend runtimes and communicate with
the authoritative server through transport Effects.

### 8.5.7 Server-authoritative ordering

When the server is authoritative, the frontend flow is:

```text
frontend intent
  -> frontend command
  -> server validates and commits
  -> server returns accepted event/state with version
  -> frontend applies accepted event
  -> projections update
```

Server responses should carry an event ID, correlation ID, and server version or
sequence. The frontend should reject, buffer, or recover from stale responses
instead of treating network arrival order as canonical.

Optimistic state should be represented separately from accepted server state:

```text
  + pending local commands/events
  -> optimistic projection
```

This lets a server rejection or replacement be applied without pretending that
the optimistic state was authoritative.

### 8.5.8 Effract commit protocol

The frontend equivalent of Livestore's local commit is:

```text
dispatch(event)
  -> enqueue event
  -> assign local sequence
  -> update(currentState, event) synchronously
  -> produce nextState and commands
  -> install event metadata and nextState
  -> increment snapshot version
  -> invalidate affected projections
  -> notify subscribers after the complete commit
  -> start commands
```

No observer should see the event without its state or the state without its
event. Commands must not mutate canonical state directly; command results must
return as later events.

For durable browser-local state, an IndexedDB transaction can append the event,
update a snapshot, and record an outbox entry together. For server-authoritative
state, the durable transaction belongs to the backend and the frontend applies
server-sequenced results.

## 9. Key Takeaways

### When Livestore's Architecture Makes Sense

- **Database-backed state** — SQLite or any embedded DB is the source of truth
- **Multi-device sync** — event sourcing + rebase protocol for conflict resolution
- **Deterministic replay** — materializers as pure functions enable time-travel and testing
- **Large state** — data lives in the DB, not in React memory; queries page through results
- **Offline-first** — local DB is primary, sync is background

### When Effract's Architecture Makes Sense

- **API-backed state** — data comes from HTTP/gRPC calls, not a local DB
- **Co-located data fetching** — REC generators describe data needs alongside rendering
- **Suspense-driven UIs** — loading states are a first-class concern
- **Server rendering** — SSR with eager effect resolution
- **Fine-grained UI state** — atoms and Computations update only dependent components
- **Application workflows** — a proposed frontend event runtime can add explicit
  transitions without giving up the atom graph

### Philosophical Differences

| | Livestore | Effract |
|---|---|---|
| **State source** | Deterministic (event log → materialized state) | Current: atoms as truth; proposed: Event -> State -> projections |
| **Sync** | Built-in protocol, rebase, rollback | Not addressed |
| **React integration** | Glue layer (effects call setState) | Native (REC generators drive renders) |
| **Learning curve** | Familiar imperative API, complex internals | Novel REC pattern, complex but consistent |
| **When it shines** | Real-time collaboration, offline-first | SSR, API-driven, rapid prototyping |

### Could They Be Combined?

The systems solve different problems and are surprisingly complementary:

- Use **Effract's** atom system and REC generators for **fine-grained UI projections** (modals, selections, form state)
- Use **Livestore's** event sourcing for **domain state** (documents, todos, collaborative data)
- Bridge via a `queryDb`-like thunk that reads canonical state and subscribes to database/table version refs
- Use a Foldkit-style frontend event runtime when explicit ordering, replay, and workflow state are valuable

The `Computation` class in Effract and the `Thunk` in Livestore are functionally equivalent — both
are derived values that track dependencies. A merged architecture could use Livestore's graph for
DB-backed state and Effract's atoms for ephemeral state, with a shared reactive bridge.

---

## 10. Appendix — Same Concept, Two Systems

### A. Derived Value from Two Atoms

**Livestore — `computed`**
```typescript
// Definition (module-level, reusable)
const filteredTodos$ = computed((get) => {
  const todos = get(todos$)         // depends on todos table
  const filter = get(filterText$)   // depends on signal
  return todos.filter(t => t.text.includes(filter))
})

// In component
const filteredTodos = useQuery(filteredTodos$)
```

**Effract — `observe`**
```typescript
// In component (or module)
const filtered = observe(($) => {
  const todos = $(todosAtom)         // depends on todosAtom
  const filter = $(filterTextAtom)   // depends on filterTextAtom
  return todos.filter(t => t.text.includes(filter))
})

function Todos() {
  const filtered = useAtomValue(filtered)
  // ...
}
```

**Key difference:** Livestore defines the derived value declaratively at module scope via `computed`.
Effract uses the `observe` helper (which creates a `Computation`). Both achieve the same result —
a value that updates when dependencies change — but via different APIs.

### B. Subscribe to Changes

**Livestore — `store.subscribe`**
```typescript
const unsubscribe = store.subscribe(todos$, (newTodos) => {
  console.log('todos changed:', newTodos.length)
})
// unsubscribe() to stop
```

**Effract — `atom.subscribe`**
```typescript
const unsubscribe = todosAtom.subscribe((newTodos) => {
  console.log('todos changed:', newTodos.length)
})
// unsubscribe() to stop
```

Both return unsubscribe functions. Livestore's `subscribe` creates a reactive graph `Effect` node;
Effract's `subscribe` adds to the atom's listener `Set`.

### C. Write/Mutate State

**Livestore — Event**
```typescript
store.commit(events.todoCreated({
  id: nanoid(),
  text: 'Buy milk',
  completed: false,
}))

// Materializer:
const todoCreated = ({ id, text, completed }) =>
  todos.insert({ id, text, completed })
```

**Effract — Direct**
```typescript
todosAtom.update(prev => [...prev, {
  id: nanoid(),
  text: 'Buy milk',
  completed: false,
}])
```

**Key difference:** Livestore encodes the intent (`todoCreated`), not the state change. The materializer
decides how to project the event. Current Effract mutates the state directly. A proposed Effract event
runtime would preserve the same distinction while using Effract Computations for fine-grained projections.
Livestore's approach additionally enables durable sequencing, syncing, replay, and audit. Direct Effract
atoms remain simpler and appropriate for ephemeral UI state.

### D. Async Query with Caching

**Livestore — N/A**

There is no direct equivalent. If you need to fetch from a remote API:

```typescript
// You'd typically commit an event that triggers a sync
store.commit(events.fetchTodosRequested())

// Or use a signal + async effect (manual)
const todos$ = signal([] as Todo[])
const [apiTodos, setApiTodos] = useState<Todo[]>([])
useEffect(() => {
  fetch('/api/todos').then(r => r.json()).then(setApiTodos)
}, [])
```

**Effract — `query`**
```typescript
const todos$ = query(
  Effect.fnUntraced(function*() {
    const response = yield* fetch('/api/todos')
    return yield* response.json()
  }),
  'todos-list' as const
)

function Todos() {
  const todos = useAtomValue(todos$)
  // Suspends while fetching, shows fallback
  // Stale-while-revalidate on refetch
}
```

**Key difference:** Effract natively supports async data fetching with caching, dedup, and suspense.
Livestore requires manual orchestration or integration with a sync protocol because its data model
assumes the database is the source of truth.

---

## Implementation References

The atomicity and writer analysis above is based on the upstream implementation:

- Repository: https://github.com/livestorejs/livestore
- `Store.commit`: `packages/@livestore/livestore/src/store/store.ts`
- Client synchronization: `packages/@livestore/common/src/sync/ClientSessionSyncProcessor.ts`
- Sync merge and rebase: `packages/@livestore/common/src/sync/syncstate.ts`
- Reactive graph: `packages/@livestore/livestore/src/reactive.ts`

The important implementation sequence is:

```text
Store.commit
  -> syncProcessor.encodeEvents
  -> sqliteDbWrapper.txn(materialize) for multi-event commits
  -> syncProcessor.push
  -> batched table-ref refresh
```

The client sync processor uses `FiberHandle` to interrupt and restart the leader
push loop during rebase, `TxQueue` to batch pushes, `SyncState` to validate event
heads and detect divergence, and SQLite changesets to roll back optimistic local
materialization before applying upstream events and rebasing pending events.
