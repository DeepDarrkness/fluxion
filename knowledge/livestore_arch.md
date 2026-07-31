# Livestore Implementation Knowledge

> Extracted from `livestore.dev` — an event-sourced, reactive state management library built on Effect-TS and SQLite.
>
> **Goal of this document:** Capture the architecture, dataflow, and key implementation patterns so we can build a
> similar library with in-memory state instead of SQLite. SQLite-specific details are summarised at a high level;
> the focus is on generalisable reactive/event-sourcing mechanics.

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Dependency Graph (`ReactiveGraph`)](#2-dependency-graph-reactivegraph)
   - 2.1 [Node Types](#21-node-types)
   - 2.2 [Edge Tracking](#22-edge-tracking)
   - 2.3 [Refresh Propagation](#23-refresh-propagation)
   - 2.4 [Deferred Effects & Batching](#24-deferred-effects--batching)
3. [Query System](#3-query-system)
   - 3.1 [Query Definitions (`LiveQueryDef`)](#31-query-definitions-livequerydef)
   - 3.2 [Reference Counting (`RcRef`)](#32-reference-counting-rcref)
   - 3.3 [`queryDb` — Reactive SQL Queries](#33-querydb--reactive-sql-queries)
   - 3.4 [`computed` — Derived Values](#34-computed--derived-values)
   - 3.5 [`signal` — Ephemeral Local State](#35-signal--ephemeral-local-state)
4. [Event Lifecycle](#4-event-lifecycle)
   - 4.1 [Commit → Encode → Materialize → Push](#41-commit--encode--materialize--push)
   - 4.2 [Event Schema & Sequence Numbers](#42-event-schema--sequence-numbers)
   - 4.3 [Materializers](#43-materializers)
   - 4.4 [Table Refs & Reactive Triggering](#44-table-refs--reactive-triggering)
5. [Sync State Machine](#5-sync-state-machine)
   - 5.1 [SyncState](#51-syncstate)
   - 5.2 [Merge Algorithm](#52-merge-algorithm)
   - 5.3 [Rebase Handling](#53-rebase-handling)
   - 5.4 [Session Changesets](#54-session-changesets)
6. [React Bridge](#6-react-bridge)
   - 6.1 [`useStore` — Suspending Store Loader](#61-usestore--suspending-store-loader)
   - 6.2 [`useQuery` — Reactive Rendering](#62-usequery--reactive-rendering)
   - 6.3 [`useRcResource` — Reference-Counted Lifecycle](#63-usercresource--reference-counted-lifecycle)
   - 6.4 [The Render Trigger Chain](#64-the-render-trigger-chain)
7. [Effect Fibers & Concurrency](#7-effect-fibers--concurrency)
   - 7.1 [Push Batching](#71-push-batching)
   - 7.2 [Background Pull Stream](#72-background-pull-stream)
   - 7.3 [Fiber Interruption during Rebase](#73-fiber-interruption-during-rebase)
8. [Adapter Interface](#8-adapter-interface)
9. [Key Files Reference](#9-key-files-reference)

---

## 1. Architecture Overview

### High-Level Dataflow

```
┌─────────────────────────────────────────────────────────────────────────┐
│  React App                                                             │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────┐  │
│  │  useStore()   │    │   useQuery()     │    │  store.commit(...)   │  │
│  │  (suspense)   │    │  (subscribe)     │    │  (events)            │  │
│  └──────┬───────┘    └────────┬─────────┘    └──────────┬───────────┘  │
│         │                     │                          │              │
└─────────┼─────────────────────┼──────────────────────────┼──────────────┘
          │                     │                          │
          ▼                     ▼                          ▼
┌─────────────────────────────────────────────────────────────────────────┐
│  Store                                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  ReactivityGraph (reactive.ts)                                    │ │
│  │  ┌──────┐    ┌──────────┐    ┌──────────┐    ┌───────────────┐   │ │
│  │  │ Ref  │───▶│  Thunk   │───▶│  Thunk   │───▶│   Effect      │   │ │
│  │  │(tbl) │    │(db.query)│    │(computed)│    │ (subscription) │   │ │
│  │  └──┬───┘    └──────────┘    └──────────┘    └───────┬───────┘   │ │
│  │     │                                                  │           │ │
│  │     │ setRefs(tableRefs)                               │ callback  │ │
│  │     │                                                  ▼           │ │
│  │     │                                          ┌──────────────┐   │ │
│  │     │                                          │ React setState│   │ │
│  │     │                                          └──────────────┘   │ │
│  │     │                                                  │           │ │
│  └─────┼──────────────────────────────────────────────────┼───────────┘ │
│        │                                                  │              │
│  ┌─────┴──────────────────────────────────────────────────┴───────────┐ │
│  │  ClientSessionSyncProcessor (sync/ClientSessionSyncProcessor.ts)  │ │
│  │  ┌──────────┐   ┌──────────────┐   ┌──────────┐   ┌──────────┐   │ │
│  │  │ encode   │──▶│ materialize  │──▶│  push    │──▶│ TxQueue  │───│─│──▶ Leader Thread
│  │  │ (seqNum) │   │ (run mat'rs) │   │ (merge)  │   │ (batch)  │   │ │
│  │  └──────────┘   └──────────────┘   └──────────┘   └──────────┘   │ │
│  │                                                                    │ │
│  │  ┌──────────────────────────────────────────────┐                 │ │
│  │  │  Background Pull Stream                      │                 │ │
│  │  │  (Stream.fromQueue → merge → materialize)    │◀────────────────│─│── Leader Thread
│  │  └──────────────────────────────────────────────┘                 │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  SQLite Database (SqliteDbWrapper)                                │ │
│  │  • State DB — tables from schema, materializer writes             │ │
│  │  • Eventlog DB — event store for sync/replay                      │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

### Key Principles

1. **Event sourcing**: All state changes happen through events. Events are immutable, sequenced, and stored in an append-only eventlog.
2. **Materialized state**: Events are projected into queryable state via materializer functions (pure, deterministic SQL operations).
3. **Reactive propagation**: A dependency graph (Refs + Thunks + Effects) eagerly propagates changes with cut-off on value equality.
4. **Optimistic concurrency**: Local commits are immediately visible; sync happens asynchronously. Conflicts are handled via rebase.
5. **Reference counting**: Query instances are reference-counted so they're shared across components and disposed when unused.

---

## 2. Dependency Graph (`ReactiveGraph`)

**File:** `packages/@livestore/livestore/src/reactive.ts` (701 lines)

The reactive graph is a standalone, framework-agnostic implemention of a reactive dependency graph
inspired by the [MiniAdapton](https://arxiv.org/abs/1609.05337) paper (though without lazy
recomputation — LiveStore evaluates eagerly).

### 2.1 Node Types

The graph has three node types:

| Node | Tag | Description |
|------|-----|-------------|
| `Ref<T>` | `'ref'` | Mutable cell. Holds a value that can be set explicitly. `isDirty` is always `false` — refs are never marked dirty, only their downstream thunks are. |
| `Thunk<T>` | `'thunk'` | Pure computation. Re-evaluates when its dependencies change. Has `isDirty` flag, `previousResult`, `equal(a, b)` for change detection. |
| `Effect` | `'effect'` | Side-effect. Runs when any of its dependencies change. Return value is ignored. Used for subscriptions (e.g., triggering React re-renders). |

### 2.2 Edge Tracking

Each node maintains two adjacency sets:

- **`sub`** (sub-computations / upstream): Atoms that this node _depends on_. For thunks/effects, this is filled during execution via the `getAtom` function.
- **`super`** (super-computations / downstream): Nodes (thunks/effects) that _depend on_ this node. For refs, only thunks/effects; for thunks, only thunks/effects.

Edges are added dynamically when a thunk or effect calls `getAtom(someAtom)` during its computation. The `addEdge` method links `superComp.sub.add(subComp)` and `subComp.super.add(superComp)`.

### 2.3 Refresh Propagation

When a `Ref` value changes via `setRef` or `setRefs`:

1. **Mark dirty**: `markSuperCompDirtyRec(ref, effectsToRefresh)` walks the `super` edges recursively:
   - For `Thunk` nodes: sets `isDirty = true`, continues walking their `super` edges.
   - For `Effect` nodes: collects into `effectsToRefresh` set (doesn't recurse further).
2. **Run effects**: `runEffects(effectsToRefresh)` iterates the collected effects and calls `effect.doEffect()`.
3. **Inside `doEffect`**: The effect resets its `sub` set (clearing old dependencies), then calls its user function which calls `getAtom()` for each dependency. `getAtom()` calls `compute(atom)`:
   - If the atom is dirty, calls `atom.computeResult()` which re-runs the thunk logic, tracks new dependencies via `addEdge()`, compares new vs old result with `equal()`, and sets `isDirty = false`.
   - If the atom is clean, returns the cached `previousResult`.
4. **Cut-off**: If `equal(old, new)` returns true, the thunk's result hasn't changed, but downstream effects still run. Cut-off only prevents re-computation of _thunks_ — effects always run when any dependency changes.

```
setRef(tableRef, null)                     commit → setRefs([tableRef1, tableRef2])
  │                                               │
  ▼                                               ▼
markSuperCompDirtyRec(ref, effectsSet)            markSuperCompDirtyRec for each ref
  │                                               │
  ├──▶ Thunk.isDirty = true                       │
  │     └──▶ markSuperCompDirtyRec(thunk, ...)    │
  │         └──▶ Effect → collected               │
  └──▶ Effect → collected                         │
                                                   ▼
                                          runEffects(effectsSet)
                                                │
                                                ├──▶ effect.doEffect()
                                                │     └──▶ get(dependency$)
                                                │           └──▶ compute(atom)
                                                │                 ├── dirty → recompute & compare
                                                │                 └── clean → return cached
                                                └──▶ runRefreshCallbacks()
```

### 2.4 Deferred Effects & Batching

Effects can be _deferred_ by passing `skipRefresh: true` to `setRefs`:

- Instead of running immediately, effects are stored in `deferredEffects: Map<Effect, Set<RefreshReason>>`.
- A later call to `runDeferredEffects()` (or `store.manualRefresh()`) runs all deferred effects.
- This allows batching multiple commits and triggering a single refresh cycle.

The `effectsWrapper` from the `ReactivityGraphContext` can wrap `runEffects` — in React, this is set to `ReactDOM.unstable_batchedUpdates` to batch React state updates.

---

## 3. Query System

**Files:**
- `packages/@livestore/livestore/src/live-queries/base-class.ts` (373 lines)
- `packages/@livestore/livestore/src/live-queries/db-query.ts` (471 lines)
- `packages/@livestore/livestore/src/live-queries/computed.ts` (166 lines)
- `packages/@livestore/livestore/src/live-queries/signal.ts` (137 lines)

### 3.1 Query Definitions (`LiveQueryDef`)

Queries are defined as lightweight definition objects (blueprints) that can be created at module scope
and shared across components:

```typescript
interface LiveQueryDef<TResult> {
  _tag: 'def' | 'signal-def'
  make: (ctx: ReactivityGraphContext, otelContext?: otel.Context) => RcRef<LiveQuery<TResult> | ISignal<TResult>>
  label: string
  hash: string          // Derived from query string or explicit deps
  [Equal.symbol](): boolean
  [Hash.symbol](): number
}
```

The `make` function creates a query _instance_ (stateful, bound to a specific Store's reactivity graph).
`make` is wrapped by `withRCMap` to provide reference-counted sharing.

### 3.2 Reference Counting (`RcRef`)

```typescript
interface RcRef<T> {
  rc: number
  value: T
  deref: () => void    // Decrements rc; destroys when rc === 0
}
```

`withRCMap(id, makeInstance)` maintains a `defRcMap: Map<string, RcRef>` per reactivity graph:

- On first `make(ctx)`: creates instance with `rc = 1`, stores in map.
- On subsequent `make(ctx)` with same `id`: increments `rc`, returns cached instance.
- On `deref()`: decrements `rc`; when `rc === 0`, calls `destroy()`, removes from map.

This ensures that multiple components using the same query definition share a single LiveQuery instance.

### 3.3 `queryDb` — Reactive SQL Queries

`queryDb` accepts either a `QueryBuilder` (typed DSL), a raw SQL string with schema, or a thunk function `(get) => QueryInput`.

Internally it creates a `LiveStoreDbQuery` which sets up **two thunks** in the reactive graph:

1. **`queryInput$`** (optional, only for thunk queries): A thunk that computes the query SQL and bind values. Its dependencies are signals or other queries referenced in the thunk.
2. **`results$`** (always): A thunk that:
   - Gets (and depends on) the query input
   - Establishes dependencies on **table refs** (parsed from the SQL string)
   - Executes the query against the database
   - Decodes results using the schema
   - Applies the `map` function if provided
   - Uses `Schema.toEquivalence(schema)` for value-equality cut-off

```
queryDb(tables.todos.where({ complete: true }))
  │
  └── LiveStoreDbQuery
        ├── queryInput$ (Thunk) — only if using (get) => ... thunk form
        │     └── deps on signals/other queries used in the thunk
        │
        └── results$ (Thunk)
              ├── deps on queryInput$ (if thunk)  OR  static query string
              ├── deps on tableReF['todos']         (from SQL parsing)
              └── compute: db.cachedSelect(sql, bind) → decode → map
```

### 3.4 `computed` — Derived Values

`computed((get) => TResult)` creates a `LiveStoreComputedQuery`:

- Single `results$` thunk that calls the user function with `makeGetAtomResult(get)`.
- The user's `get()` calls inside the fn establish reactive dependencies.
- Memoized: only re-evaluates when dependencies change; uses `===` equality by default, so primitive values get cut-off.

### 3.5 `signal` — Ephemeral Local State

`signal(defaultValue)` creates a `SignalDef` → when made, creates a `Signal` instance:

- Wraps a `Ref` in the reactivity graph via `reactivityGraph.makeRef(defaultValue)`.
- `store.setSignal(signalDef, value)` → signal's `ref.set(value)` → triggers downstream propagation.
- Signals are not persisted or synced — purely local, ephemeral state.
- Useful for UI state (search filters, selections) that should drive query re-evaluation.

---

## 4. Event Lifecycle

### 4.1 Commit → Encode → Materialize → Push

```
store.commit(event1, event2, ...)

  1. checkShutdown
  2. getCommitArgs → flatten events, extract options
  3. Effect.gen:
     a. syncProcessor.encodeEvents(events)
          └── assigns client sequence numbers (Composite seq num)
          └── resolves SessionIdSymbol → concrete sessionId
          └── returns EncodedWithMeta[]
     b. syncProcessor.materializeEvents(encodedEvents)
          └── for each event: getExecStatementsFromMaterializer(...)
          └── runs materializer SQL against sqliteDbWrapper
          └── captures sessionChangeset (for rollback during rebase)
          └── collects writeTables (which tables were modified)
       └── Wrapped in db transaction if > 1 event
     c. syncProcessor.push(encodedEvents)
          └── SyncState.merge({ local-push, ... })
          └── updates syncStateRef
          └── queues events to leaderPushQueue (TxQueue)
          └── leader pushing fiber batches and sends to leader thread
     d. reactivityGraph.setRefs(tableRefs, { debugRefreshReason, skipRefresh })
          └── for each modified table, set the table Ref → triggers reactive propagation
```

### 4.2 Event Schema & Sequence Numbers

Events use **composite sequence numbers** (`EventSequenceNumber.Client.Composite`):

- `seqNum` is a client-local logical clock (monotonically increasing per session).
- `rebaseGeneration` tracks how many times this session's events have been rebased.
- `isClientOnly` flag distinguishes events that should never be synced.

The `EncodedWithMeta` class wraps encoded event data with mutable metadata:
- `sessionChangeset` for rollback support
- `materializerHashSession` / `materializerHashLeader` for consistency checking

### 4.3 Materializers

Materializers are pure functions defined by the user:
```typescript
type Materializer<TEventDef> = (
  event: TEventDef['schema']['Type'],
  context: { query, eventDef, currentFacts, event }
) => MaterializerResult | MaterializerResult[]
```

Where `MaterializerResult` is either:
- A `QueryBuilder` (typed DSL) — e.g. `todos.insert({ id, text })`
- A raw SQL string
- `{ sql, bindValues, writeTables? }` object

The `getExecStatementsFromMaterializer` helper decodes the event, invokes the materializer, and
produces executable SQL statements with bind values.

### 4.4 Table Refs & Reactive Triggering

The Store creates one `Ref<null>` per table in the schema (lines 352-372 in `store.ts`):

```typescript
for (const tableName of allTableNames) {
  tableRefs[tableName] = reactivityGraph.makeRef(null, {
    equal: () => false,   // Always propagates (every commit is a change)
    label: `tableRef:${tableName}`,
  })
}
```

These refs use `equal: () => false` — every table ref update unconditionally propagates.
When `commit` materializes events, it collects all `writeTables` and calls
`reactivityGraph.setRefs(tableRefs)` which sets each affected table ref.

DB queries (`LiveStoreDbQuery`) read the table refs they depend on during their `results$` thunk
execution, establishing edges: `ref → thunk → effect`.

---

## 5. Sync State Machine

**Files:** `packages/@livestore/common/src/sync/syncstate.ts` (613 lines), `ClientSessionSyncProcessor.ts` (423 lines)

### 5.1 SyncState

```typescript
class SyncState {
  pending: EncodedWithMeta[]          // Events pending upstream acknowledgment
  upstreamHead: Composite             // Last event confirmed by upstream
  localHead: Composite                // Last event this session has produced
}
```

Invariants:
1. Chain continuity: each event references its immediate parent via sequence number
2. `upstreamHead ≤ localHead`
3. Events follow the pattern e1 → e1.1 → e1.2 → e2

### 5.2 Merge Algorithm

The `merge` function processes 3 payload types:

| Payload | Trigger | Effect |
|---------|---------|--------|
| `local-push` | Local commit | Append pending events, advance localHead |
| `upstream-advance` | Leader sent new events (no conflict) | Advance upstreamHead, confirm matching pending events |
| `upstream-rebase` | Leader sent new events + rollback (conflict) | Roll back conflicting pending events, apply new events, rebase unconfirmed events on top |

Each returns a `MergeResult`:
- `Advance` — clean advance, no rollback needed
- `Rebase` — requires rolling back some pending events before applying new ones
- `Reject` — upstream rejected the push (e.g., sequence number mismatch)

### 5.3 Rebase Handling

When `MergeResult` is `Rebase`:

```
1. Interrupt the leader pushing fiber  (FiberHandle.clear)
2. Clear the leader push queue         (TxQueue.clear)
3. For each rollbackEvent (in reverse order):
     rollback(event.meta.sessionChangeset.data)  ← undoes SQLite changes
4. Offer remaining pending events back to leaderPushQueue
5. Restart the leader pushing fiber  (FiberHandle.run)
```

The rollback uses the SQLite changeset captured during materialization (`sessionChangeset`).

### 5.4 Session Changesets

When events are materialized, the session changeset captures the SQLite diff:

```typescript
sessionChangeset =
  | { _tag: 'sessionChangeset'; data: Uint8Array<ArrayBuffer>; debug: any }
  | { _tag: 'no-op' }
  | { _tag: 'unset' }
```

This is only captured when `withChangeset: true` (the pull/materialize path). During local commits
(`withChangeset: false` by default), changesets are not captured for commit events — they're only
captured when events later come back through the pull path during sync.

---

## 6. React Bridge

**Files:**
- `packages/@livestore/react/src/useStore.ts` (115 lines)
- `packages/@livestore/react/src/useQuery.ts` (163 lines)
- `packages/@livestore/react/src/useRcResource.ts` (201 lines)
- `packages/@livestore/framework-toolkit/src/query.ts` (164 lines)

### 6.1 `useStore` — Suspending Store Loader

```typescript
const useStore = (options) => {
  const storeRegistry = useStoreRegistry()
  const storeOrPromise = storeRegistry.getOrLoadPromise(options)
  const store = storeOrPromise instanceof Promise
    ? React.use(storeOrPromise)    // Suspends while loading
    : storeOrPromise
  React.useEffect(() => storeRegistry.retain(options), [storeRegistry, options])
  return withReactApi(store)
}
```

Key design decisions:
- `getOrLoadPromise` is called **every render** (not memoized) to avoid `React.use()` with resolved promises blocking transitions.
- `React.use()` (React 19's Suspense primitive) enables synchronous fallback after resolution.
- `retain()` is called in `useEffect` (after `React.use()`) to avoid hook-order violations during Suspense.

### 6.2 `useQuery` — Reactive Rendering

The React → Store bridge works in several phases:

**Phase 1: Normalization & Resource Creation**
```typescript
const normalized = normalizeQueryable(queryable)   // wraps QueryBuilder → queryDb(def)
const rcRefKey = computeRcRefKey(store, normalized) // unique key for RC caching
const { queryRcRef, span, otelContext } = createQueryResource(store, normalized, stackInfo)
```

**Phase 2: Initial Run**
```typescript
const initialResult = runInitialQuery(query$, otelContext, stackInfo, 'react')
// Creates a synchronous useState-style ref:
const [valueRef, setValue] = useStateRefWithReactiveInput<TResult>(initialResult)
```

**Phase 3: Subscription**
```typescript
React.useEffect(() => {
  return store.subscribe(query$, (newValue) => {
    if (deepEqual(newValue, valueRef.current) === false) {
      setValue(newValue)  // Triggers React re-render
    }
  })
}, [...])
```

The `store.subscribe` call creates an `Effect` node in the reactivity graph:
```typescript
const effect = reactivityGraph.makeEffect((get, _, debugRefreshReason) => {
  const result = get(query$.results$, otelContext, debugRefreshReason)
  if (!suppressCallback) onUpdate(result)
})
```

When any table ref changes, the effect fires, recomputes the query result, and calls the React `setValue`.

### 6.3 `useRcResource` — Reference-Counted Lifecycle

This is a generic hook for reference-counted resource management:

```
useRcResource(scope, key, create, dispose, options?)
```

- Maintains a `WeakMap<scope, Map<key, { rc, resource }>>` cache.
- On first render with a key: calls `create()`, stores with `rc = 1`.
- On concurrent renders with same key: increments `rc`, returns cached resource.
- On key change or unmount: decrements `rc`, calls `dispose()` when `rc === 0`.
- Handles React Strict Mode double-mounting via `didDisposeInMemo` flag.

### 6.4 The Render Trigger Chain

The full chain from event commit to React re-render:

```
store.commit(events)
  → materializer SQL writes
  → collect writeTables = { 'todos', 'users', ... }
  → reactivityGraph.setRefs([tableRef_todos, tableRef_users])
    → markSuperCompDirtyRec(tableRef_todos, effectsSet)
      → marks results$ thunk dirty
        → marks subscription effect dirty
    → runEffects(effectsSet)
      → subscription effect.doEffect()
        → get(results$)  → compute(results$)
          → get(tableRef_todos) → marks dep
          → db.cachedSelect(...) → new results
          → decode, map
        → onUpdate(newResults)
          → setValue(newResults)  ← React state update
            → component re-renders
```

---

## 7. Effect Fibers & Concurrency

**File:** `packages/@livestore/common/src/sync/ClientSessionSyncProcessor.ts` (423 lines)

### 7.1 Push Batching

Local commits are not pushed to the leader immediately — they're batched via a `TxQueue`:

```typescript
const leaderPushQueue = yield* TxQueue.unbounded<EncodedWithMeta>()

const backgroundLeaderPushing = TxQueue.takeBetween(leaderPushQueue, 1, batchSize)
  .pipe(Effect.flatMap(batch => leaderThread.push(batch)))
  .pipe(Effect.forever, Effect.forkScoped)
```

`TxQueue.takeBetween(queue, min, max)` collects events as they arrive, up to `max=100` per batch,
with a minimum of 1 (so it still goes through if there's only 1 event). This is forked as a background
fiber managed by `FiberHandle`.

### 7.2 Background Pull Stream

A separate fiber runs the pull loop:

```typescript
Stream.suspend(() => leaderThread.events.pull({ cursor: syncStateRef.current.upstreamHead }))
  .pipe(
    Stream.tap(payload => Effect.gen(function* () {
      const mergeResult = yield* SyncState.merge({ syncState, payload, ... })
      // Handle advance or rebase...
      // Materialize new events
      // refreshTables(writeTables)
    })),
    Stream.runDrain,
    Effect.forever
  )
```

The pull stream continuously receives events from the leader, merges them into the local sync state,
materializes new events, and refreshes table refs to trigger reactive updates.

### 7.3 Fiber Interruption during Rebase

When a rebase happens:

```typescript
// 1. Interrupt the push fiber
yield* FiberHandle.clear(leaderPushingFiberHandle)

// 2. Clear the push queue (abort pending pushes)
yield* TxQueue.clear(leaderPushQueue)

// 3. Roll back conflicting events (in reverse order)
for (let i = rollbackEvents.length - 1; i >= 0; i--) {
  rollback(rollbackEvents[i]!.meta.sessionChangeset.data)
}

// 4. Re-queue remaining pending events
yield* TxQueue.offerAll(leaderPushQueue, newSyncState.pending)

// 5. Restart the push fiber
yield* FiberHandle.run(leaderPushingFiberHandle, backgroundLeaderPushing)
```

This ensures the push fiber doesn't send stale events while rebasing, and cleanly restarts with the
rebased pending events.

---

## 8. Adapter Interface

**File:** `packages/@livestore/common/src/adapter-types.ts` (152 lines)

The `Adapter` is the platform-specific entry point:

```typescript
type Adapter = (args: AdapterArgs) => Effect<ClientSession, UnknownError, Scope.Scope>

interface ClientSession {
  sqliteDb: SqliteDb
  clientId: string
  sessionId: string
  leaderThread: ClientSessionLeaderThreadProxy
  lockStatus: SubscriptionRef<LockStatus>
  shutdown: (cause: Exit<...>) => Effect<void>
}
```

Three implementations exist (for our in-memory version, we'd follow the `makeInMemoryAdapter` pattern):

| Adapter | Location | Persistence | Concurrency |
|---------|----------|-------------|-------------|
| `makeInMemoryAdapter` | `adapter-web/src/in-memory/` | None (RAM) | Leader + session in same process |
| `makeSingleTabAdapter` | `adapter-web/src/single-tab/` | OPFS / localStorage | Single tab |
| `makePersistedAdapter` | `adapter-web/src/web-worker/` | OPFS via SharedWorker | Multi-tab (SharedWorker leader) |

To build an in-memory-only version, the key substitution is replacing `SqliteDb` with a
plain JavaScript data structure that implements the same interface (`prepare`, `select`, `exec`, `txn`, etc.).

---

## 9. Key Files Reference

### Core Library (`packages/@livestore/livestore/src/`)

| File | Purpose |
|------|---------|
| `reactive.ts` | Reactive dependency graph (Refs, Thunks, Effects, propagation) |
| `store/store.ts` | Store class: commit, subscribe, query, setSignal |
| `store/store-types.ts` | StoreInternals, RefreshReason, SyncStatus, type guards |
| `store/create-store.ts` | `createStore()` Effect factory |
| `store/StoreRegistry.ts` | StoreRegistry with RC-based lifecycle |
| `live-queries/base-class.ts` | LiveQuery/LiveQueryDef/RcRef/GetAtomResult, `withRCMap` |
| `live-queries/db-query.ts` | `queryDb()` — reactive SQL queries, `LiveStoreDbQuery` |
| `live-queries/computed.ts` | `computed()` — derived values, `LiveStoreComputedQuery` |
| `live-queries/signal.ts` | `signal()` — ephemeral state, `Signal` |
| `SqliteDbWrapper.ts` | SQLite wrapper with caching, table detection |
| `QueryCache.ts` | Result cache keyed by SQL+bind, invalidated by table |

### Common Package (`packages/@livestore/common/src/`)

| File | Purpose |
|------|---------|
| `sync/syncstate.ts` | SyncState class, merge algorithm, merge results |
| `sync/ClientSessionSyncProcessor.ts` | Encode/materialize/push orchestration, pull stream |
| `schema/EventDef/materializer.ts` | Materializer type, `defineMaterializer()`, `materializers()` |
| `schema/EventDef/define.ts` | `defineEvent()`, `synced()`, `clientOnly()` |
| `schema/schema.ts` | `LiveStoreSchema`, `makeSchema()`, `getEventDef()` |
| `schema/state/sqlite/table-def.ts` | `table()` definition, type-level helpers |
| `schema/state/sqlite/client-document-def.ts` | `clientDocument()` tables with optimistic set |
| `schema/state/sqlite/make-state.ts` | `makeState()` — builds state from tables + materializers |
| `schema/LiveStoreEvent/` | Event encoding types: Client, Global, Input, EncodedWithMeta |
| `adapter-types.ts` | Adapter interface, ClientSession, StorageMode |

### React Bindings (`packages/@livestore/react/src/`)

| File | Purpose |
|------|---------|
| `useStore.ts` | `useStore()` — suspending store loader, `withReactApi()` |
| `useQuery.ts` | `useQuery()`, `useQueryRef()` — reactive query hooks |
| `useRcResource.ts` | Reusable reference-counted resource management |
| `useClientDocument.ts` | `useClientDocument()` — React-state-like document hook |
| `StoreRegistryContext.tsx` | React context provider for StoreRegistry |

### Framework Toolkit (`packages/@livestore/framework-toolkit/src/`)

| File | Purpose |
|------|---------|
| `query.ts` | `normalizeQueryable()`, `computeRcRefKey()`, `runInitialQuery()`, `createQueryResource()` |
| `types.ts` | Shared types for framework agnostic query hooks |
| `client-document.ts` | Validation and helpers for client document tables |

---

## Appendix: Data Flow Diagram (Text)

```
User Interaction
       │
       ▼
store.commit(event)
       │
       ├──▶ syncProcessor.encodeEvents()
       │     └── assign composite sequence numbers
       │
       ├──▶ syncProcessor.materializeEvents()
       │     └── for each event:
       │           ├── resolveEventDef → find materializer
       │           ├── getExecStatementsFromMaterializer → SQL
       │           ├── execute SQL on db (inside txn if >1 event)
       │           └── capture sessionChangeset (for rollback)
       │
       ├──▶ syncProcessor.push()
       │     └── SyncState.merge({ local-push })
       │     └── TxQueue.offerAll(pushQueue, pendingEvents)
       │           └── Background fiber eventually sends to leader
       │
       └──▶ reactivityGraph.setRefs(tableRefs)
             └── markSuperCompDirtyRec for each table ref
             └── runEffects(subscription effects)
                   └── each effect:
                         ├── get(tableRef)       ← establishes edge
                         ├── get(query$.results$) ← recomputes if dirty
                         │     └── db.cachedSelect
                         └── onUpdate(newResult)
                               └── React setState
                                     └── component re-renders
```
