# Unitflow Implementation Knowledge

> Source analysis of `timurrakhimzhan/unitflow` at commit
> `020b3e075f6e38f5112dfc77f60236fc9f3618cc`.
>
> Unitflow is an Effect-native frontend dataflow library. Its core runtime is a
> registry-local collection of stores, events, streams, model instances, and
> Effect scopes. React is an external-store renderer bridge rather than the
> owner of application state or asynchronous work.

## 1. Architecture Overview

Unitflow's primary public programming model is a model containing typed stores,
events, queries, mutations, child models, and a renderer-facing `ui` shape.
`Model.Service` defines an Effect service whose `make` program runs once per
model key and returns the model's ports. The model's `inputs` and `outputs` are
application wiring surfaces; `ui` is the surface that a renderer binds to.

The core primitives are intentionally separate:

| Primitive | Responsibility |
|---|---|
| `Store` | Current renderable state and derived state. |
| `Event` | A typed occurrence/action channel. |
| `Query` | Model-owned asynchronous reads represented by `AsyncResult`. |
| `Mutation` | Serialized asynchronous writes with progress and completion state. |
| `Model` | Effect service, dependency-injection boundary, ownership boundary, and instance lifetime. |
| `UnitflowRuntime` | Synchronous snapshots and subscriptions for a renderer. |

The central runtime is not a reducer loop and does not maintain one global
application model. Stores and events are descriptors with registry-local
backing resources. A `Registry` is shared by the Effect runtime, while model
instances are independently keyed and independently scoped. See
`packages/core/src/registry.ts:78-105` and
`packages/core/src/model.ts:523-587`.

### High-Level Dataflow

```text
React event handler
        |
        v
runtime.emit(event, value)
        |
        v
Event.dispatchUnsafe / Store.writeUnsafe
        |
        +-------------------- synchronous publication ------------------+
        |                                                               |
        v                                                               v
event/store subscribers                                      direct handler queues
        |                                                               |
        v                                                               v
Effect stream fiber                                      owner-scoped handler fiber
        |                                                               |
        +---------------------- Store.set/update -----------------------+
                                      |
                                      v
                         derived-store recomputation
                                      |
                                      v
                          UnitflowRuntime snapshot
                                      |
                                      v
                         useSyncExternalStore listener
                                      |
                                      v
                              React render/commit
```

Publication itself is synchronous, but stream consumption and runtime snapshot
updates are performed by Effect fibers. The runtime deliberately counts
publications before waking subscribers so `Registry.allSettled` can wait for
the complete cascade. See `packages/core/src/store.ts:294-310`,
`packages/core/src/event.ts:267-286`, and
`packages/core/src/registry.ts:145-155`.

Effect is visible to application authors. Model `make` programs are ordinary
Effect programs and can use services, layers, streams, schedules, typed
errors, scopes, and finalizers. Effect is simultaneously the application
programming model, the orchestration runtime, and the lifecycle mechanism; it
is not merely hidden behind a signal API. The model service type and layer
construction are implemented in `packages/core/src/model.ts:643-705`.

## 2. Core State Representation

### 2.1 Plain Stores

`Store.make(initial)` creates a process-local descriptor containing an ID,
initial value, and capability markers. The descriptor is not itself the
current value. The full private descriptor has both source and sink
capabilities; public `Output` and `Input` types narrow whether it can be read
or written. See `packages/core/src/store.ts:44-82`.

The actual value is a registry-local Effect `SubscriptionRef`. It is created
lazily, indexed by store ID, and finalized with the owner scope. Closing that
scope removes the registry entry, closes listeners, and shuts down the backing
PubSub. See `packages/core/src/store.ts:150-171`.

```typescript
// Source example, equivalent to the public API.
const count = Store.make(0)
const current = yield* Store.get(count)
yield* Store.set(count, current + 1)
```

The descriptor is therefore reusable only inside the lifetime of the registry
and owning model instance that materialized it. A model eagerly materializes
all ports in its returned shape, including private sources reachable through
derived values, so those resources belong to the model instance scope. See
`packages/core/src/model.ts:495-521`.

### 2.2 Derived Stores

`Store.combine` creates a read-only graph node containing source descriptors
and a compute function. It owns no `SubscriptionRef`; `get` recursively pulls
the current source values and applies the function. Its `initial` value is
computed from source initials to make pre-subscription snapshots consistent.
See `packages/core/src/store.ts:84-121`.

```typescript
// Source example, from the Store API shape.
const doubled = count.pipe(Store.map((value) => value * 2))
const summary = Store.combine([count, doubled], (value, twice) => ({ value, twice }))
```

Flattened stores, used by `Model.list.select`, are also computed rather than
backed by a ref. They evaluate an outer collection and then select a store for
each item. See `packages/core/src/store.ts:220-231` and
`packages/core/src/model.ts:940-949`.

### 2.3 State Identity and Equality

Writes are not automatically suppressed by equality. `Store.set` publishes
the supplied value even when it is `Object.is`-equal to the old value. Equality
suppression is applied to derived streams and `Store.changed`, where repeated
computed values are filtered with `Object.is`. See
`packages/core/src/store.ts:659-675` and `packages/core/src/store.ts:768-785`.

There is no deep structural equality for values. Structural equality is used
for model instance keys by Effect's `RcMap`, not for store values. Model key
identity and structural key restrictions are documented in
`packages/core/src/registry.ts:16-30`.

## 3. State Mutation and Transition Lifecycle

### 3.1 Read Path

`Store.get` first attempts a synchronous graph pull when running inside a fiber
with a `Registry`. Plain backing refs are read directly. Combined and flattened
nodes are recursively evaluated, with a per-pull memo table for shared derived
nodes. If a required backing ref is not materialized, the operation falls back
to an Effectful path that creates refs and finalizers. See
`packages/core/src/store.ts:196-235` and `packages/core/src/store.ts:344-386`.

This is a pull-based derived model, not a permanently cached computation graph.
Derived values are recomputed on reads and on source stream emissions. The
stream layer maintains subscriptions to source leaves; it does not install a
global dependency tracker that dynamically records reads.

### 3.2 Write Path

The hot write path is synchronous when the backing ref is already materialized
and its semaphore is uncontended. Otherwise the same operation is serialized
through the ref semaphore. `Store.update` and `Store.modify` perform the
read-modify-write while holding that critical section. See
`packages/core/src/store.ts:312-342` and `packages/core/src/store.ts:388-447`.

The actual publication sequence is:

1. Open an optional debug causality window.
2. Increment the registry settlement ledger for active subscribers.
3. Set the `SubscriptionRef` value and publish to its PubSub.
4. Offer the value to direct store listeners.
5. Close the debug window.

This sequence is implemented in `packages/core/src/store.ts:294-310`.
Counting before publication is important: a subscriber woken during the
publication already has its work represented in `Registry.allSettled`.

```typescript
// Source example.
yield* Store.update(count, (value) => value + 1)
```

The callback passed to `update` executes synchronously in the write critical
section. If it throws, the Effect operation fails as a defect; there is no
store-level validation or automatic recovery layer. State schemas can be used
by application code and persistence boundaries, but ordinary `Store.set` does
not validate values at runtime. See `packages/core/src/store.ts:409-424`.

### 3.3 Events and Transition Cascades

Events are descriptors with registry-local unbounded PubSubs. A normal event
can be emitted only through an input-capable port. Setter events are backed by
a store and delegate directly to `Store.set`; combined events are source-only
merged views and cannot be emitted. See `packages/core/src/event.ts:116-199`
and `packages/core/src/event.ts:295-317`.

Event dispatch follows the analogous sequence:

```text
track publication -> publish to PubSub -> offer direct handler queues
```

See `packages/core/src/event.ts:267-286`.

`Event.handler` uses a dedicated queue and drain fiber for each handler. The
default is sequential handling. The queue prevents recursive handler execution
and preserves event order for that handler. With
`{ concurrency: "unbounded" }`, each handler invocation is forked into the
owner scope. Separate handlers on one event are independent. See
`packages/core/src/event.ts:507-555` and `packages/core/src/event.ts:558-623`.

```typescript
// Source example.
const increment = yield* Event.input<number>().pipe(
  Event.handler((amount) => Store.update(count, (value) => value + amount)),
)
```

### 3.4 Transactions, Batching, and Replay

Unitflow has no explicit transaction object, reducer batch, or event-sourced
log. A single synchronous `Store.update` is serialized, and a cascade can be
waited on through `Registry.allSettled`, but multiple writes are not committed
as one atomic snapshot for renderers. `Registry.allSettled` waits for pending
subscriber work and cascades, not for a transaction boundary. See
`packages/core/src/registry.ts:328-349`.

The debug inspector records a bounded causal event history, but this is
diagnostic history rather than a replayable durable transition log. Its
recording interface is in `packages/core/src/debug/inspector.ts:86-203`.

## 4. Reactivity and Dependency Tracking

### 4.1 Declared Graphs

Dependencies are declared by constructing derived stores, combined events,
query dependency records, and flattened model-list selections. A combined
store stores its source array explicitly. `Store.map` is simply a one-source
`combine`. See `packages/core/src/store.ts:91-132`.

For an ordinary combined store, `Store.stream` recursively flattens the source
graph, removes duplicate source IDs, watches each leaf, recomputes the derived
value, and applies `Stream.changes`. See `packages/core/src/store.ts:659-675`
and `packages/core/src/store.ts:708-713`.

There is no dynamic dependency discovery by executing a selector and recording
which stores it read. The graph is known from descriptors. This makes source
subscription management explicit and predictable, but means arbitrary reads
inside a compute callback are not tracked as dependencies.

### 4.2 Flattened Dependencies

Flattened stores have a second-level graph. The outer collection is watched,
and each selected inner store receives its own scoped watcher. When the
composition changes, removed watchers are closed before the new composition
is used. Duplicate selected IDs are tracked with multiplicity. See
`packages/core/src/store.ts:451-568`.

This is the main place where Unitflow separates invalidation from recomputation:
composition or source writes invalidate the private flattened snapshot, the
watcher synchronously pulls the current graph, and the snapshot is published
only after recomputation. The private snapshot publication is itself counted
for settlement.

### 4.3 `Store.changed`

For ordinary and combined stores, `Store.changed` uses a fused direct listener.
It materializes the leaf refs, evaluates the derived graph on every source
write, suppresses `Object.is`-equal results, and dispatches an event directly.
The current value is initialized before listener registration, so initial
materialization is not reported as a change. See
`packages/core/src/store.ts:715-812`.

Flattened stores use stream watchers instead because composition changes need
two-level subscription management. See `packages/core/src/store.ts:730-755`.

### 4.4 Cycles

The source does not provide cycle detection for store or event graphs. A
cycle created by handlers or forwarding can repeatedly publish until the
application stops it or the runtime is interrupted. `Registry.allSettled`
waits for quiescence and therefore cannot resolve while a cycle continues to
publish. This follows from the explicit forwarding and handler paths in
`packages/core/src/event.ts:366-383` and
`packages/core/src/registry.ts:328-349`.

## 5. Effect Integration

### 5.1 Registry Runtime

`UnitflowRuntime.make` creates an Effect `ManagedRuntime` and merges in
`Registry.layer`. The registry contains store refs, event channels, model
constructors, model instances, lifetime policies, instance scopes, lease
counts, and the settlement ledger. See `packages/core/src/runtime.ts:71-75`
and `packages/core/src/registry.ts:257-300`.

There is one registry per `UnitflowRuntime`, but multiple model instances and
multiple independent owner scopes inside it. The registry is shared
application infrastructure; model state and long-lived pipelines are isolated
by `InstanceScope`.

### 5.2 Fibers

Effect fibers are used for:

- Ongoing event and store pipelines through `Registry.run`.
- Direct event handler drains.
- Unbounded handler invocations.
- Query requests.
- Query pagination requests.
- React runtime store subscriptions.
- Model construction and model resolution.

`Registry.run` forks its stream drain into the required owner scope with
`startImmediately: true`. See `packages/core/src/registry.ts:304-325`.

Queries fork each request immediately and join it before publishing success or
failure. See `packages/core/src/query.ts:88-111`. Mutations use a semaphore
and run the handler within the model construction context, so callers do not
need to provide the model's services. See `packages/core/src/mutation.ts:42-72`.

### 5.3 Scopes and Layers

Models are Effect services registered through layers. The layer registers a
constructor and lifetime policy but does not eagerly run `make`; construction
occurs on the first `Model.get`. See `packages/core/src/model.ts:643-705`.

Each constructed model gets a dedicated instance scope forked from the
`RcMap` entry scope. That scope is placed in the Effect context both as
`InstanceScope` and as the ambient ordinary `Scope`, so model resources,
stores, event channels, handlers, and child leases share the model lifetime.
See `packages/core/src/model.ts:536-577`.

`ownerScope` falls back to registry scope for one-time materialization such as
store refs and PubSubs. `requiredOwnerScope` refuses that fallback for ongoing
pipelines, preventing repeated calls outside a model from leaking application-
lifetime fibers. See `packages/core/src/registry.ts:379-405`.

### 5.4 Effect Services and Typed Boundaries

Model handlers can yield application services from the model's layer. Query
handlers preserve their Effect environment and typed error channel. Mutation
handlers are captured with the construction context and expose a typed
request/response `Mutation.call` operation. The core does not require a
particular error encoding; typed Effect failures are preserved until a
feature chooses to store them in `AsyncResult` or log them.

## 6. Scheduler and Timing Model

Unitflow's scheduler is primarily the combination of synchronous JavaScript
critical sections and Effect's fiber scheduler. It does not implement a
separate priority scheduler.

### 6.1 Exact Timing Sequence

For a direct store write:

1. `Store.set` or `Store.update` enters the synchronous hot path when possible.
2. The settlement ledger is incremented.
3. The ref value and PubSub are updated synchronously.
4. Direct listeners and derived `changed` listeners run in the same publication
   section.
5. Stream consumers are resumed through Effect fiber machinery.
6. A runtime store-stream fiber updates its cached snapshot and invokes React
   listeners.
7. React schedules external-store work and renders according to React's own
   scheduler.

The write section is `packages/core/src/store.ts:294-342`. Runtime notification
is `packages/core/src/runtime.ts:100-121`.

For an event, dispatch is synchronous through the PubSub and direct handler
queue. The handler body is executed by the handler drain fiber, sequentially
unless explicitly configured as unbounded concurrency. See
`packages/core/src/event.ts:275-317` and `packages/core/src/event.ts:519-555`.

### 6.2 Queues, Coalescing, and Cancellation

Unitflow uses:

- Per-handler JavaScript arrays as event queues.
- Per-store stream arrays and deferred waiters.
- Effect `PubSub` channels.
- Effect `Deferred` values for parked consumers.
- Effect semaphores for serialized writes and mutations.

It does not use a library-level mailbox, timer debounce, rAF coalescing,
priority queue, or background priority. Event handlers can be made concurrent
with `{ concurrency: "unbounded" }`; ordinary handlers are sequential.

Store writes are not coalesced before publication. Derived streams suppress
equal computed values after recomputation. Query loads are serialized through
the refresh handler; pagination ignores `loadMore` while waiting. There is no
request key deduplication or general stale-result token. Query and mutation
fibers are interrupted by their owner scope, but a completed request does not
carry an independent version check before publishing. See
`packages/core/src/query.ts:125-157` and `packages/core/src/query.ts:190-223`.

### 6.3 Settlement

`Registry.allSettled` is a deterministic quiescence primitive. It runs supplied
triggers sequentially, then waits until counted publications have been
delivered and handled. Store replay values are explicitly uncounted. Stream
pulls confirm the previous batch on the next pull; direct handlers confirm an
item after its handler completes. See `packages/core/src/registry.ts:196-245`
and `packages/core/src/registry.ts:328-349`.

This is useful for Effect tests and causality inspection, but it is not a
render-frame scheduler and does not imply that React has committed a render.

## 7. React or Renderer Bridge

### 7.1 Store Subscription

`useStore` supplies `subscribeStore` and `getStore` to
`React.useSyncExternalStore`. The runtime's first snapshot tries a synchronous
`Store.get`, allowing a write that happened before the first subscription to
be observed instead of reverting to `store.initial`. See
`packages/core/src/runtime.ts:77-98` and
`packages/react/src/react.tsx:16-25`.

On the first listener, the runtime starts one `Store.stream` fiber. Each value
updates the cached snapshot and notifies all React listeners. When the last
listener leaves, the fiber is interrupted. See
`packages/core/src/runtime.ts:100-121`.

### 7.2 Model Subscription

`useModel` and the root `Unitflow` component use the same external-store
pattern, but their snapshot is a tagged `ModelResult`:

```text
Building -> Ready(model ports)
        \-> Failed(cause)
```

Each React subscription gets a binding scope and a model lease. Unmount closes
that scope. The model may remain alive under its idle TTL, allowing Strict Mode
remounts and quick navigation remounts to reuse state. See
`packages/core/src/runtime.ts:139-216`.

The runtime installs a death-watch finalizer on the model instance. If the
instance is disposed or expires, the cached `Ready` snapshot is replaced by
`Building` before a later remount can observe disposed ports. See
`packages/core/src/runtime.ts:189-209`.

### 7.3 View Binding

`View.make` receives a resolved unit rather than looking up a model from JSX.
`useBoundUi` turns store descriptors into current values and input events into
callbacks; nested child units pass through unchanged. `inputs` and `outputs`
are not exposed to the view callback. See
`packages/react/src/react.tsx:121-166`.

The binding implementation calls hooks while iterating `Object.entries(ui)`.
The UI record therefore has to retain a stable key set for the lifetime of the
binding component. The model construction pattern creates a fixed UI object,
but dynamically changing UI keys would violate React hook ordering. See
`packages/react/src/react.tsx:145-150`.

### 7.4 Effects During Render

Effect programs are not executed from the render callback. React render reads
runtime snapshots; event callbacks call `runtime.emit`, which starts an Effect
program through `runtime.runFork`. See `packages/react/src/react.tsx:27-33`
and `packages/core/src/runtime.ts:219-224`.

## 8. Async Data, Queries, and Caching

### 8.1 Query State

`Query.make` owns a writable `AsyncResult` store. It starts with
`AsyncResult.initial(true)`, creates a refresh event handler, attaches store
dependency streams, waits until dependency subscriptions are established, and
then emits the initial refresh. This ordering ensures dependency replay does
not trigger duplicate initial loads and that the initial load is visible to
`Registry.allSettled`. See `packages/core/src/query.ts:125-157`.

Each load:

1. Forks the request with `startImmediately: true`.
2. Marks the query waiting while retaining the previous value.
3. Joins the request fiber.
4. Stores success or failure-with-previous.

See `packages/core/src/query.ts:88-111`.

```typescript
// Source-shaped example.
const results = yield* Query.make({
  stores: { search },
  handler: ({ search }) => api.search(search),
})

return {
  inputs: { refresh: results.refresh },
  outputs: { state: results.state },
  ui: { state: results.state, refresh: results.refresh },
}
```

### 8.2 Pagination

`Query.makeInfinite` keeps the cursor in a private store and exposes a flat
array of loaded items. `loadMore` exits when the query is waiting, has no
loaded value, or has no next cursor. Successful pages update the cursor before
appending data; failures preserve the prior data and leave the cursor
retryable. See `packages/core/src/query.ts:166-225`.

### 8.3 Refresh, Retry, and Cancellation

`Query.refetchOn` connects event sources to refresh. `Query.repeat` maps an
Effect `Schedule` to repeated refresh events. Query handler failures are
represented in state and do not fail the long-lived query pipeline. See
`packages/core/src/query.ts:244-260` and `packages/core/src/query.ts:311-320`.

Refresh handling is sequential through `Event.handler`, so ordinary refreshes
do not overlap. A query request is a child fiber and is interrupted when its
owner model scope closes. There is no independent keyed query cache or stale
response generation counter in the query implementation. The main stale-work
protection is serialized refresh handling and scope interruption.

### 8.4 Persistence and Cache Scope

Query persistence is stale-while-revalidate. A stored success seeds the query
while the initial request remains waiting. A settled live result wins over a
late restore. Only settled successes are saved; decode, storage, and TTL
failures are treated as warnings or cache misses. See
`packages/core/src/query.ts:262-309` and
`packages/core/src/persistence.ts:23-66`.

Store persistence hydrates inline before returning the store and starts its
save pipeline afterward, dropping the replayed current value. See
`packages/core/src/store.ts:905-935`.

There is no intrinsic Unitflow query cache. The implementation explicitly
delegates keyed caching to Effect's `Effect.cachedWithTTL` or `Cache.make`
inside the handler. Therefore cache identity, capacity, eviction, and sharing
are application decisions rather than query-runtime behavior.

## 9. Lifecycle and Resource Management

### 9.1 Model Ownership

`Model.get` leases a shared instance in the caller's ambient scope. The
instance key is the model ID plus key, and `RcMap` provides structural key
equality for supported immutable data keys. Singleton models default to
infinite lifetime; keyed models default to a ten-minute idle TTL. See
`packages/core/src/registry.ts:16-30`, `packages/core/src/registry.ts:107-134`,
and `packages/core/src/model.ts:678-703`.

When the last lease is released, the idle policy controls disposal. A new lease
within the grace period reuses the instance and its stores. `Model.dispose`
invalidates the instance and closes its scope immediately, even if leases
remain; those old ports become inert and a later `get` constructs fresh state.
See `packages/core/src/model.ts:627-638`.

### 9.2 Resource Acquisition

Model code can use `Effect.addFinalizer`, `Effect.acquireRelease`, streams, and
other scoped Effect APIs. Because `Scope.Scope` is provided as the model's
ambient context during `make`, resources acquired there are released when the
instance scope closes. See `packages/core/src/model.ts:555-577`.

```typescript
// Simplified from the model lifetime contract.
const socket = yield* openSocket
yield* Effect.addFinalizer(() => socket.close)
```

Stores and events register finalizers that remove registry entries and shut
down their channels. Ongoing pipelines are forked into the same owner scope.
See `packages/core/src/store.ts:157-170`,
`packages/core/src/event.ts:188-197`, and
`packages/core/src/registry.ts:304-325`.

### 9.3 Child Models and Dynamic Lists

`Model.list` owns child leases through dedicated sub-scopes. Removing a child
first publishes the new list composition and only then closes the child scope.
This ordering allows flattened selectors to switch away from the removed child
before its store channels shut down. A child is disposed immediately after its
list lease is released only if no other holder remains. See
`packages/core/src/model.ts:803-950`.

### 9.4 Abandoned Renders and Strict Mode

React render itself does not acquire models. The external-store subscription
does. This means an abandoned render that never commits does not own a model
lease; the subscription lifecycle controls ownership. A committed subscription
is released on unsubscribe. Strict Mode's mount/unmount/remount cycle is
handled by the binding scope and keyed model idle TTL, while the death watch
prevents stale disposed snapshots. See
`packages/core/src/runtime.ts:147-216`.

The source does not implement a separate render-claim or reference count tied
to speculative React renders. Ownership is subscription-scope based, with
`RcMap` leases mirrored by the registry.

## 10. Error Handling

### 10.1 State Writes and Derived Computation

`Store.set` has an `E = never` type and does not validate values. A throwing
update callback becomes an Effect defect. Combined-store compute functions are
ordinary JavaScript functions; a thrown exception similarly defects the pull
or stream computation. There is no store-level error state. See
`packages/core/src/store.ts:388-447`.

### 10.2 Event Pipelines

Handler APIs constrain the handler's error channel to `never`. Expected
interruptions and PubSub shutdowns terminate normally. Unexpected defects
terminate the affected pipeline and are logged rather than failing the whole
registry runtime. See `packages/core/src/event.ts:610-618` and
`packages/core/src/registry.ts:247-255`.

`Mutation` catches failures for fire-and-forget event delivery and writes
failure-with-previous into its state. `Mutation.call` preserves the typed
failure channel for callers. See `packages/core/src/mutation.ts:55-96`.

### 10.3 Queries and Persistence

Query handler failures are stored as `AsyncResult` failure state while
previous successful data remains available. Persistence failures are logged
and swallowed, and malformed or expired values are treated as misses. See
`packages/core/src/query.ts:93-111` and
`packages/core/src/persistence.ts:41-66`.

### 10.4 Model Construction and React

Model construction failures invalidate the `RcMap` entry before waking
waiters, close the partially-created scope, and are exposed to the renderer as
`ModelResult.Failed`. React's `Unitflow` and keyed views accept optional
`failed` render callbacks; the default is no rendered output. See
`packages/core/src/model.ts:578-585`,
`packages/core/src/runtime.ts:166-187`, and
`packages/react/src/react.tsx:74-112`.

Errors are not automatically sent to React Error Boundaries or Suspense. They
remain Effect causes, query/mutation state, pipeline logs, or explicit model
construction render state according to the feature involved.

## 11. React Scheduler Compatibility

### 11.1 Side-by-Side Timing

```text
Unitflow dataflow:
  write
    -> synchronous ref/event publication
    -> direct listener or Effect stream/handler fiber
    -> runtime cached snapshot update
    -> external-store notification

React external store:
  external-store notification
    -> React schedules work
    -> getSnapshot during render
    -> render may restart or be abandoned
    -> commit if current
```

Unitflow commits its dataflow transition before React renders. React never
speculatively runs the Unitflow model `make` program as part of rendering.
React reads the latest runtime snapshot through `getSnapshot`. See
`packages/core/src/runtime.ts:94-121` and
`packages/react/src/react.tsx:16-25`.

### 11.2 Snapshot Consistency and Tearing

The bridge is designed for `useSyncExternalStore`: snapshots are cached in the
runtime, `getSnapshot` returns the cached value, and model snapshots preserve
object identity when the same live instance is resolved again. This avoids
unnecessary snapshot changes and supports React's consistency checks. See
`packages/core/src/runtime.ts:171-187`.

The store bridge is not a transaction coordinator. Multiple synchronous writes
can publish multiple values before React commits. React may render the latest
snapshot, while Effect subscribers may have processed intermediate values.
That is intentional for dataflow, but consumers requiring one render-level
atomic view must update one store or derive one combined snapshot.

### 11.3 Interruption and Abandonment

React can interrupt or abandon render work, but Unitflow's data transitions are
not rolled back. Since effects execute from event handlers, model construction,
and owner-scoped pipelines rather than render callbacks, abandoned React work
does not leave an Effect fiber created during render. Subscription cleanup is
performed when React unsubscribes, and the binding scope closes the model lease.
See `packages/core/src/runtime.ts:213-216`.

### 11.4 Transitions and Background Work

Unitflow does not call `startTransition`, expose a priority API, or classify
updates as urgent/background. An application can wrap a React-side response
in a transition, but the underlying Unitflow publication and Effect work are
not reprioritized by that transition. Likewise, `concurrency: "unbounded"`
means concurrent Effect fibers, not lower-priority background work.

### 11.5 Strict Mode and Cleanup

Strict Mode remounts produce separate binding scopes and leases. Singleton or
keyed model identity is retained according to the model lifetime policy. The
default keyed idle TTL protects quick remounts; zero-TTL or explicit disposal
closes the instance, and the death-watch resets `Ready` to `Building` before a
new instance is exposed. See `packages/core/src/registry.ts:107-110` and
`packages/core/src/runtime.ts:189-209`.

## 12. Detailed Dataflow Diagrams

### 12.1 Store Write to React

```text
Store.update(inputStore, f)
        |
        v
read SubscriptionRef under semaphore
        |
        v
f(current) -> next
        |
        v
trackPublish(store.id)
        |
        +--> SubscriptionRef value + PubSub publication
        |             |
        |             +--> Store.stream consumers
        |                          |
        |                          +--> derived recomputation
        |                          +--> query dependency reload
        |                          +--> runtime StoreEntry.value
        |                                      |
        |                                      +--> React listeners
        |
        +--> direct Store.changed listeners
                      |
                      +--> Event.dispatchUnsafe
```

Implementation anchors: `packages/core/src/store.ts:388-424`,
`packages/core/src/store.ts:659-675`,
`packages/core/src/store.ts:758-812`, and
`packages/core/src/runtime.ts:100-121`.

### 12.2 Event Handler

```text
runtime.emit(event, value)
        |
        v
registry-local channel lookup
        |
        v
trackPublish -> PubSub.publishUnsafe -> offerHandlers
                                      |
                                      v
                              handler queue.push(value)
                                      |
                                      v
                              handler drain fiber wakes
                                      |
                       +--------------+--------------+
                       |                             |
                 sequential                    unbounded
                       |                             |
                 handle(value)             forkIn(handle(value))
                       |                             |
                       +-------------> completeCounted
```

Implementation anchors: `packages/core/src/event.ts:275-317` and
`packages/core/src/event.ts:507-555`.

### 12.3 Query Refresh

```text
dependency store change / refresh event / schedule tick
        |
        v
query refresh handler
        |
        v
read fresh dependency values
        |
        v
forkChild(request, startImmediately)
        |
        +--> state = waiting(previous)
        |
        v
join request
        |
        +--> success(value) -> state success(value)
        |
        +--> failure(cause) -> state failureWithPrevious(cause)
```

Implementation anchors: `packages/core/src/query.ts:93-157`.

### 12.4 Model and Resource Lifetime

```text
React subscription / parent Model.get
        |
        v
RcMap.get(model, key) -> one shared instance
        |
        v
instance scope + InstanceScope context
        |
        +--> stores and events
        +--> handlers and streams
        +--> queries and mutations
        +--> child model leases
        +--> Effect finalizers
        |
unsubscribe / parent disposal / idle TTL / runtime disposal
        |
        v
close scope -> interrupt fibers -> remove registry resources
```

Implementation anchors: `packages/core/src/model.ts:536-623`,
`packages/core/src/model.ts:627-638`, and
`packages/core/src/registry.ts:257-300`.

## 13. Reusable Ideas

### 13.1 Separate Descriptor, State, and Capability

**Idea:** Make a state primitive a typed descriptor while keeping current state
in a runtime-local resource.

**Why it matters:** It allows `Output` and `Input` capabilities to be exposed
independently and lets ownership determine when actual state exists.

**Source:** `packages/core/src/store.ts:44-79`.

```typescript
const count = Store.make(0)
// expose count as Output, keep the full Store privately for writes
```

**Area:** Effect dataflow.

**Tradeoffs:** Descriptors are not self-contained values and cannot be used
without the correct registry/runtime. Debugging identity also becomes
process-local and descriptor-ID based.

### 13.2 Pull-Based Derived Values with Explicit Source Graphs

**Idea:** Store derived nodes as source descriptors plus a compute function,
then pull current values and separately build streams for invalidation.

**Why it matters:** It keeps reads synchronous and makes derived values easy to
use as external-store snapshots without maintaining a second cache.

**Source:** `packages/core/src/store.ts:84-121`, `packages/core/src/store.ts:196-235`.

```typescript
const visible = Store.combine([items, filter], (items, filter) =>
  items.filter((item) => item.includes(filter)),
)
```

**Area:** Effect dataflow, React integration.

**Tradeoffs:** Compute work can repeat across reads, dependencies must be
declared explicitly, and cycles are not detected.

### 13.3 Fused Synchronous Change Detection

**Idea:** For ordinary derived stores, attach direct source listeners that
recompute and publish only when `Object.is` changes.

**Why it matters:** Simple derived state avoids an extra asynchronous watcher
pipeline and preserves causality inside the originating write.

**Source:** `packages/core/src/store.ts:758-812`.

```typescript
const changed = yield* Store.changed(visible)
yield* changed.pipe(Event.handler((next) => log(next)))
```

**Area:** Scheduling, Effect dataflow.

**Tradeoffs:** The listener must synchronously evaluate the graph, and the
special flattened-store path needs more complicated scoped watchers.

### 13.4 Count Before Publish

**Idea:** Increment settlement accounting before waking subscribers.

**Why it matters:** Synchronous cascades cannot outrun the accounting ledger,
so deterministic `allSettled` can include work triggered by the publication.

**Source:** `packages/core/src/registry.ts:145-155` and
`packages/core/src/store.ts:294-310`.

```text
track pending -> publish -> handler/stream work -> complete pending
```

**Area:** Scheduling, Testing.

**Tradeoffs:** The ledger is complex, must distinguish replay from counted
publications, and represents quiescence rather than React commit completion.

### 13.5 Direct Event Handler Queues

**Idea:** Use a per-handler queue for direct event handling instead of forcing
every handler through a generic stream combinator.

**Why it matters:** Subscription registration can happen synchronously at fork,
preventing same-chain emissions from being lost, while preserving sequential
order by default.

**Source:** `packages/core/src/event.ts:201-245` and
`packages/core/src/event.ts:507-623`.

```typescript
yield* saveRequested.pipe(
  Event.handler((request) => save(request)),
)
```

**Area:** Effect dataflow, Scheduling.

**Tradeoffs:** There are two subscription implementations to maintain, and
unbounded concurrency changes completion semantics and can reorder effects.

### 13.6 One Owner Scope for Ongoing Work

**Idea:** Require an explicit instance scope for repeated pipelines while
allowing registry fallback only for one-time resource materialization.

**Why it matters:** It prevents repeated setup calls from silently forking
application-lifetime fibers and gives model unmount/disposal a deterministic
cleanup boundary.

**Source:** `packages/core/src/registry.ts:379-405`.

```typescript
yield* Registry.run(source.pipe(Stream.mapEffect(handle)))
```

**Area:** Effect dataflow, Lifecycle/resource management.

**Tradeoffs:** Some APIs cannot be used outside a model without
`InstanceScope.root`, and scope requirements become part of the public Effect
environment.

### 13.7 Model Instances as Leased Effect Services

**Idea:** Resolve keyed models through a structural `RcMap`, then mirror leases
with explicit scope finalizers.

**Why it matters:** Parents, React bindings, lists, and tests can share one
instance while retaining deterministic ownership and idle retention.

**Source:** `packages/core/src/model.ts:595-623` and
`packages/core/src/registry.ts:112-134`.

```typescript
const child = yield* Model.get(TaskModel, { id: taskId })
```

**Area:** Lifecycle/resource management, React integration.

**Tradeoffs:** Structural keys must remain immutable, idle retention complicates
memory expectations, and explicit disposal can invalidate still-held ports.

### 13.8 Stable External-Store Snapshots

**Idea:** Cache runtime snapshots and preserve their object identity when the
underlying model has not changed.

**Why it matters:** `useSyncExternalStore` compares snapshots by identity and
requires coherent committed reads under concurrent rendering.

**Source:** `packages/core/src/runtime.ts:77-98` and
`packages/core/src/runtime.ts:171-187`.

```typescript
const value = React.useSyncExternalStore(subscribe, getSnapshot, getSnapshot)
```

**Area:** React integration.

**Tradeoffs:** Snapshot caching adds a second runtime-facing representation and
must be reset when the underlying model scope dies.

### 13.9 Death-Watch Stale Snapshot Prevention

**Idea:** Attach a finalizer to a live model scope that resets the renderer
snapshot before disposed ports can be reused.

**Why it matters:** Idle TTL expiration and explicit disposal otherwise leave a
cached `Ready` object pointing at dead resources.

**Source:** `packages/core/src/runtime.ts:189-209`.

```text
scope closes -> if cached model is the dead instance -> Ready becomes Building
```

**Area:** React integration, Lifecycle/resource management.

**Tradeoffs:** The bridge needs a race-safe identity guard and an additional
notification path for resource death.

### 13.10 Query State as Ordinary Writable State

**Idea:** Represent async query status as a writable `AsyncResult` store rather
than a renderer-specific hook state.

**Why it matters:** Loading, previous data, failure, optimistic updates,
persistence, and React rendering all use the same store/dataflow machinery.

**Source:** `packages/core/src/query.ts:93-111` and `packages/core/src/query.ts:125-157`.

```typescript
const query = yield* Query.make({ handler: () => api.load() })
yield* Store.get(query.state)
```

**Area:** Effect dataflow, Caching.

**Tradeoffs:** The query does not supply intrinsic key deduplication or stale
response generation checks; applications must add caching or stronger request
identity when needed.

### 13.11 Scope-Based Dynamic Child Collections

**Idea:** Give each dynamic child entry a sub-scope and publish collection
composition before closing removed entries.

**Why it matters:** It prevents flattened selectors from observing late writes
from removed children and allows shared children to survive one owner's removal.

**Source:** `packages/core/src/model.ts:828-918`.

```typescript
const rows = yield* Model.list(RowModel)
```

**Area:** Lifecycle/resource management, Effect dataflow.

**Tradeoffs:** Entry bookkeeping and finalizer ordering are subtle, especially
when construction races or multiple owners share a child.

### 13.12 Deterministic Effect Quiescence

**Idea:** Provide `Registry.allSettled` as a causality-aware alternative to
arbitrary sleeps in tests and orchestration code.

**Why it matters:** It waits through multi-stage event/store cascades while
excluding uncounted initial replays.

**Source:** `packages/core/src/registry.ts:328-349`.

```typescript
yield* Registry.allSettled(Event.emit(increment, 1))
```

**Area:** Testing, Scheduling.

**Tradeoffs:** It can wait forever for a productive cycle, and completion says
nothing about whether React has rendered or committed the resulting snapshot.

## 14. Risks, Tradeoffs, and Open Questions

### Scheduler and Rendering

- There is no explicit priority model connecting Effect work to React lanes.
- React transitions can defer rendering but cannot reprioritize already-started
  store writes or Effect fibers.
- A burst of synchronous writes can produce many dataflow publications before
  React commits a render.
- `Registry.allSettled` is not a render synchronization primitive.

### Async Correctness

- Query refreshes are serialized, but the query abstraction has no general
  request-key or generation-token stale-result guard.
- Query and mutation cancellation depends primarily on Effect interruption and
  owner-scope disposal.
- Query caching and eviction are delegated to application-provided Effect
  caches, so cache scope is not standardized by Unitflow.

### React Hook Constraints

- `useBoundUi` uses one hook per UI record entry. Dynamic UI key sets are unsafe.
- The bridge uses the same callback for client and server snapshots; applications
  with meaningful SSR data differences may need an additional hydration design.
- Runtime event emission is fire-and-forget. React callers needing completion
  must use a separate Effect-facing API such as `Mutation.call` or
  `Registry.allSettled`.

### Lifecycle Semantics

- Explicit `Model.dispose` can invalidate ports still held by other leases.
  This is intentional, but callers must treat disposal as a force-close.
- Idle TTL retention improves remount behavior but can retain state and
  resources longer than a component tree suggests.
- Structural model keys must be immutable after insertion because Effect's hash
  and equality machinery may cache object identity information.

### Graph Semantics

- Store dependencies are explicit, which is predictable but less flexible than
  dynamically traced selectors.
- Cycles are not detected at graph construction time.
- Equality is `Object.is`, so mutable objects require immutable publication
  discipline if consumers depend on change notifications.

### Open Questions for a React-plus-Effect Design

1. Should React-facing APIs expose an explicit transition/background policy,
   or should all priority decisions remain in the renderer layer?
2. Should async resources carry generation identities in addition to scope
   interruption to prevent late results from publishing?
3. Should a future runtime expose an atomic combined snapshot API for multiple
   stores, or rely on `Store.combine` and external-store identity rules?
4. Should dynamic UI binding use a fixed schema or move from hook-per-entry
   binding to one external-store subscription per model unit?
5. Should model disposal be forbidden while external leases remain, or should
   force disposal continue to be an explicit escape hatch?

## 15. Key Files Reference

| File | Role |
|---|---|
| `packages/core/src/store.ts` | Store descriptors, refs, synchronous writes, derived stores, streams, `changed`, persistence, and `waitFor`. |
| `packages/core/src/event.ts` | Event descriptors, PubSubs, synchronous dispatch, handler queues, forwarding, and event waiting. |
| `packages/core/src/model.ts` | Effect service definitions, lazy construction, leases, instance scopes, disposal, and dynamic child lists. |
| `packages/core/src/registry.ts` | Registry service, owner scopes, settlement accounting, model maps, and pipeline execution. |
| `packages/core/src/runtime.ts` | Managed Effect runtime, renderer snapshots, store/model subscriptions, and fire-and-forget emission. |
| `packages/core/src/query.ts` | Async query state, dependency reloads, pagination, persistence, refetch, and schedules. |
| `packages/core/src/mutation.ts` | Serialized async writes, mutation state, typed calls, completion events, and invalidation. |
| `packages/core/src/persistence.ts` | Shared schema-backed persistence adapter and best-effort load/save behavior. |
| `packages/react/src/react.tsx` | `useSyncExternalStore` hooks, runtime provider, root/model leasing, and UI binding. |
| `packages/core/src/debug/inspector.ts` | Bounded causal event history, store snapshots, instance lifecycle, and debug attachment. |
| `packages/core/test/registry.test.ts` | Settlement accounting, publication cascades, replay handling, and forwarding behavior. |
| `packages/core/test/handler-concurrency.test.ts` | Same-chain subscription registration and concurrent handler semantics. |
| `packages/core/test/model-runtime.test.ts` | Snapshot identity, model construction, leases, remount reuse, and stale snapshot reset. |
| `packages/core/test/query.test.ts` | Query dependencies, schedules, pagination, persistence, and failure behavior. |
| `packages/core/test/ownership.test.ts` | Model leases, idle lifetimes, list ownership, and disposal behavior. |
| `packages/react/test/view.test.tsx` | View binding types, fixed UI surfaces, unit ownership, and headless-model rejection. |
