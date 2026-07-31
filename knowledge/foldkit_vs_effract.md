# Foldkit Ideas for Effract

## Architecture Design Report

This report evaluates which ideas from Foldkit's Elm-inspired architecture can
improve an Effract-style React and Effect library. It also places those ideas in
the context of the Elm Architecture, CQRS, event sourcing, Livestore, and
Effract's current reactivity implementation.

The goal is not to reproduce Foldkit or to enumerate differences. The goal is
to identify a better architecture for a React UI library in which:

- React owns rendering and reconciliation.
- Effect owns dataflow, asynchronous work, resources, and lifecycle.
- React components can remain declarative and composable.
- Local UI state can remain fine-grained and reactive.
- Domain state can optionally use event sourcing and CQRS techniques.

---

## High-Level View

The proposed Effract architecture combines Foldkit's explicit Elm-style
transition loop, Livestore's Event -> State materialization boundary, Effract's
fine-grained reactive graph, and React's rendering scheduler.

```text
                           BACKEND
             Separate Effect Runtime and Scope Tree
                              |
                HTTP / WebSocket / transport events
                              |
                              v
┌────────────────────────────────────────────────────────────┐
│              FRONTEND EFFECT APPLICATION RUNTIME            │
│                                                            │
│  Root application Scope                                    │
│    - event queue and single transition processor            │
│    - command fibers and subscriptions                      │
│    - HTTP/WebSocket resources                              │
│    - feature scopes and lifecycle                          │
│                                                            │
│  Event Model                                                │
│    React intent / transport event / timer / port            │
│             |                                              │
│             v                                              │
│    ordered Event -> pure update/materializer                │
│             |                                              │
│             +----------------------+                       │
│             |                      |                       │
│             v                      v                       │
│    atomic canonical State     Effect Commands               │
│             |                      |                       │
│             v                      v                       │
│    Effract reactive graph    result Events                  │
│      - derived projections       |                          │
│      - dependency tracking       +--------------------------│
│      - fine-grained invalidation                            │
│             |                                               │
│             v                                               │
│    projected state snapshots                             │
└─────────────┬──────────────────────────────────────────────┘
              |
              v
┌────────────────────────────────────────────────────────────┐
│                         REACT                               │
│  View RECs read projected state                             │
│  useSyncExternalStore provides coherent snapshots           │
│  React schedules, interrupts, defers, and commits rendering  │
└────────────────────────────────────────────────────────────┘
```

The central invariant is:

> All meaningful frontend state transitions enter one Effect-owned event
> runtime. That runtime atomically commits event sequence, canonical state,
> projection invalidation, and snapshot version. View RECs read projected state
> during React render; React decides when that state becomes visible.

This is not a module-global singleton. It is one application runtime per mount
or request, with a root Effect scope. The frontend and backend Effect runtimes
are separate systems connected by serialized transport events.

The architectural split is intentional:

```text
Effect runtime = truth, causality, commands, resources, lifecycle
Effract graph  = derived state, invalidation, fine-grained projections
View REC       = declarative projected-state reader
React          = render scheduling and commit
```

Foldkit provides the explicit event-to-state mental model. Livestore provides
durable event sequencing, materialization, and invalidation ideas where domain
state requires them. Effract retains the graph and React bridge instead of
owning a competing renderer.

---

## 1. Executive Recommendation

Effract should evolve toward a hybrid architecture with two complementary
dataflow modes.

### Reactive atom mode

Use atoms and derived atoms for small, local, high-frequency state:

```text
event handler -> atom.set/update -> derived computation -> React snapshot
```

Examples include input drafts, popover visibility, selected tabs, drag state,
and transient animation state.

### Feature runtime mode

Use typed messages, deterministic updates, Effect commands, and subscriptions
for feature workflows and domain state:

```text
UI event / external input
  -> Message
  -> update(Model, Message)
  -> committed Model snapshot
  -> Commands and subscriptions
  -> result Message
  -> next update
  -> React snapshot
```

### Durable domain mode

Use event sourcing and CQRS only for domains that benefit from durable history,
conflict handling, synchronization, auditability, or independently optimized
read models:

```text
Command
  -> validation and domain decision
  -> durable DomainEvent
  -> materializer / projection
  -> reactive read model
  -> React snapshot
```

The important boundary is that a UI `Message`, an Effect `Command`, and a
durable domain `Event` are not automatically the same thing.

---

## 2. Terminology and Boundaries

These terms should remain distinct in the public architecture.

| Concept | Meaning | Durable? | Typical owner |
|---|---|---:|---|
| Intent | Something a user or host wants to happen | No | React or host |
| Message | A typed input to a feature update loop | Usually no | Feature runtime |
| Command | An Effect program requested by an update | No | Effect runtime |
| Domain event | An accepted fact about a domain change | Often yes | Domain/write model |
| Model | Current materialized feature state | Usually no | Feature runtime |
| Read model | State shaped for efficient queries or rendering | Rebuildable | Query/projection side |
| Derived state | A value computed from current state | No | Reactive graph |
| Snapshot | Stable renderer-facing value | No | React adapter |

For example:

```text
User clicks "Book"
  -> Intent: BookClicked
  -> Message: BookingRequested
  -> Command: SubmitBooking
  -> Domain decision: BookingAccepted
  -> Domain event: BookingCreated
  -> Read model: booking status = confirmed
```

In a simple local feature, the message may directly update the model. In an
event-sourced domain, the message should normally cause a command, and the
resulting accepted domain event should be what changes the durable state.

---

## 3. Elm Architecture

The Elm Architecture consists of three core functions and a runtime:

```text
Model --view--> UI
  ^              |
  |              v
  +-- update <-- Message
```

In practical form:

```ts
type Update<Model, Message> = (
  model: Model,
  message: Message,
) => readonly [Model, ReadonlyArray<Command<Message>>]

type View<Model, Message> = (model: Model) => ViewTree<Message>
```

The Elm Guide describes the Model as application state, the View as a function
from state to HTML, and Update as the function that changes state in response to
messages. Commands and Subscriptions extend this model to external effects.

### 3.1 Why the pattern is useful

The most useful property is not the shape of the functions. It is the explicit
causal chain:

```text
input -> message -> state transition -> effect description
```

That makes several behaviors visible:

- Every state transition has an identifiable cause.
- Side effects are not hidden inside view code.
- Asynchronous results re-enter through the same message path.
- A feature can be tested without rendering or executing Effects.
- The state transition function can be replayed when its inputs are known.
- External inputs can be normalized into one internal protocol.

### 3.2 Commands

A Command is not the result of an Effect. It is a value describing Effect work
that will eventually produce a Message.

```ts
type Command<Message, Error = never, Env = never> =
  Effect<Message, Error, Env>
```

The runtime should execute returned commands outside `update` and dispatch their
results back into the message queue.

```ts
const update = (model: Model, message: Message) => {
  switch (message._tag) {
    case "LoadRequested":
      return [
        { ...model, loading: true },
        [loadTodos.pipe(Effect.map((todos) => TodosLoaded({ todos })))]
      ] as const
  }
}
```

Commands should support explicit policies rather than one universal behavior:

- `concurrent`: all invocations run independently.
- `serial`: invocations run in arrival order.
- `latest`: interrupt or ignore earlier work when a newer request arrives.
- `keyed(key)`: only work for the same logical key conflicts.
- `debounced`: wait for a quiet period before starting.
- `retrying`: apply an Effect `Schedule`.

Effect already supplies fibers, interruption, queues, schedules, scopes, and
semaphores. Effract needs a policy layer that maps command metadata to those
primitives.

### 3.3 Subscriptions

A Subscription is a model-dependent external stream. Its lifecycle is derived
from the current model rather than from an individual React component.

```ts
const chatSubscription = Subscription.make({
  dependencies: (model: Model) => ({ room: model.roomId }),
  stream: ({ room }) =>
    room === undefined
      ? Stream.empty
      : Chat.events(room).pipe(Stream.map(ChatMessageReceived)),
})
```

The runtime should:

1. Compute initial dependencies.
2. Start the initial stream.
3. Recompute dependencies after each committed model change.
4. Compare dependencies using an explicit equivalence.
5. Interrupt the old stream when requirements change.
6. Start the replacement stream.
7. Dispatch stream values as Messages.

This is particularly valuable for WebSockets, browser events, route changes,
polling, visibility changes, and Livestore synchronization.

### 3.4 Ports

Foldkit's typed Ports are a useful host boundary for an Effract runtime. Ports
should validate data at the boundary and translate it into internal Messages.

```ts
const HostSearch = Port.inbound(SearchRequestSchema)
const SearchChanged = Port.outbound(SearchResultSchema)
```

The host should not receive the internal Model or an unrestricted dispatch
function. It should receive a narrow, schema-validated protocol.

---

## 4. CQRS and Event Sourcing

CQRS means Command Query Responsibility Segregation: the model used to change
information is separated from the model used to read information. CQRS does not
require event sourcing, separate databases, or messaging, although those are
common additions.

### 4.1 CQRS levels

CQRS can be introduced in increasing levels of separation:

#### Level 1: Separate interfaces

The same state is exposed through separate write and read capabilities.

```ts
type ProductCommands = {
  rateProduct: (input: RateProduct) => Effect<void, DomainError>
}

type ProductQueries = {
  byId: (id: ProductId) => Readonly<ProductView> | undefined
}
```

This is a low-cost way to prevent views from mutating state directly.

#### Level 2: Separate in-memory models

The write model validates commands and produces state changes. The read model
contains projection-shaped data for views.

```text
Command model -> projection -> query model -> React
```

#### Level 3: Separate stores

The write model and read model use different persistence or indexing strategies.
This is useful when read and write workloads have different scaling, security, or
schema requirements.

#### Level 4: Event-sourced CQRS

The write side persists accepted domain events. Read models are materialized from
those events.

```text
event log -> one or more projections -> query stores
```

### 4.2 Benefits for Effract

CQRS can improve Effract in the following ways:

- **Intent-oriented APIs:** `BookRoom` is more meaningful than `setStatus`.
- **Mutation safety:** Components receive commands rather than writable atoms.
- **Read optimization:** React can consume view-shaped snapshots without domain
  objects or write-side invariants.
- **Security boundaries:** Write capabilities and read capabilities are explicit.
- **Collaboration:** Commands can carry enough intent to support conflict policy.
- **Testing:** Command handling and query projections can be tested independently.
- **Persistence flexibility:** Local UI state can remain in memory while domain
  state uses a durable event log.

### 4.3 Costs and failure modes

CQRS is not a default state-management pattern. Its costs include:

- More types and translation layers.
- Duplicate representations of related data.
- More complicated loading and failure states.
- Eventual consistency between command and query models.
- Duplicate delivery and retry handling.
- More difficult debugging when a projection lags.
- Schema and migration work for commands, events, and read models.
- A tendency to create meaningless CRUD commands such as `SetName` everywhere.

Martin Fowler's CQRS guidance is especially important here: apply CQRS to
specific complex bounded contexts, not automatically to an entire application.
For simple CRUD-like state, a shared model is usually easier and safer.

### 4.4 Event sourcing is a separate decision

Event sourcing stores all accepted state changes as an ordered event stream. The
current state is a projection of that stream.

```ts
type Event =
  | { _tag: "TodoCreated"; id: string; text: string }
  | { _tag: "TodoCompleted"; id: string }

const apply = (state: State, event: Event): State => {
  switch (event._tag) {
    case "TodoCreated":
      return addTodo(state, event)
    case "TodoCompleted":
      return completeTodo(state, event.id)
  }
}
```

The benefits are:

- Complete history.
- Rebuildable projections.
- Temporal queries.
- Deterministic replay, assuming deterministic handlers.
- New projections without changing the original event log.
- Audit and support workflows.
- Conflict resolution and synchronization foundations.

The costs are substantial:

- Event schemas become long-lived compatibility contracts.
- Old events must remain readable or be migrated.
- Projections need rebuild and checkpoint strategies.
- External calls cannot be blindly repeated during replay.
- External query results may need to be recorded.
- Ordering, idempotency, duplicates, and partial failure must be handled.
- A long event stream needs snapshots or incremental projections.

Foldkit's bounded DevTools history is therefore not event sourcing. It can
reconstruct debug state by replaying messages, but it does not make those
messages a durable source of truth.

---

## 5. How Elm's Model Resembles Event-Sourced State

The resemblance is real but incomplete.

| Foldkit/Elm concept | Event-sourced analogue | Important difference |
|---|---|---|
| Message | Input or command | May be transient UI intent |
| `update` | Event application/projector | May process requests, not accepted facts |
| Model | Materialized state | Usually in-memory and replaceable |
| Command | Effect request | Not necessarily a domain event |
| DevTools history | Debug replay | Bounded and non-durable |
| Subscription | External event source | Often produces UI messages, not domain facts |

The safe architecture is:

```text
UI message
  -> feature update
  -> Effect command
  -> domain service
  -> accepted domain event
  -> projection
  -> feature/read-model message or snapshot
```

Do not persist every UI Message by default. For example, `SearchInputChanged`
should normally not become a durable event, while `InvoiceIssued` probably
should.

---

## 6. Effract's Current Reactivity

The current Effract design has several strong foundations:

- Atoms use Effect's `AtomRef` for storage.
- Atom writes use Effect equality and suppress equal values.
- Subscribers use `useSyncExternalStore`.
- `Computation` dynamically tracks dependencies by tracing reads.
- Derived computations subscribe lazily and release dependencies when unused.
- Queries have runtime-scoped cache identity, deduplication, stale-while-
  revalidate behavior, cancellation, and claims-based cleanup.
- RECs provide a generator-based render-time programming model.
- Async Effects integrate with React Suspense through `React.use()`.
- Runtime and component scopes are handled explicitly.

These capabilities should not be discarded in favor of a global reducer. They
solve different problems from Elm's message loop.

### 6.1 Current propagation path

The current atom path is approximately:

```text
atom.set(value)
  -> Equal.equals(new, old)
  -> store value
  -> notify listeners immediately
  -> recompute derived computations
  -> notify dependent listeners
  -> React useSyncExternalStore
```

This is effective for local state, but every source of change has essentially
the same urgency. Typing, a WebSocket message, cache invalidation, and polling
can all enter the same immediate notification path.

### 6.2 Main architectural limitation

The current `Computation` combines several responsibilities:

- dependency discovery
- dependency subscription
- invalidation
- recomputation
- equality cutoff
- downstream notification

That is simple, but makes it difficult to introduce scheduling policies without
changing the graph implementation.

A more extensible design separates the stages:

```text
Dependency graph
  -> invalidator
  -> scheduler
  -> recomputation engine
  -> equality policy
  -> React notification adapter
```

---

## 7. Proposed Effract Dataflow

The recommended runtime should have an explicit committed-state pipeline.

```text
React event / port / subscription / command result
                    |
                    v
                Message queue
                    |
                    v
          deterministic update function
                    |
          +---------+----------+
          |                    |
          v                    v
    committed Model       Effect Commands
          |                    |
          v                    v
  invalidate projections   scoped fibers
          |                    |
          v                    v
   stable read snapshot <- result Messages
          |
          v
  useSyncExternalStore
          |
          v
        React
```

### 7.1 Message runtime

Use a runtime-local queue rather than a module-global queue.

```ts
interface FeatureRuntime<Model, Message> {
  readonly getSnapshot: () => Model
  readonly subscribe: (listener: () => void) => () => void
  readonly dispatch: (message: Message) => void
  readonly dispatchEffect: (
    effect: Effect<Message, unknown, unknown>,
  ) => void
  readonly dispose: () => Effect<void>
}
```

The queue should provide:

1. FIFO ordering by default.
2. A processing flag to prevent recursive updates.
3. Batch snapshots so new messages cannot starve older ones.
4. A disposed guard that drops late messages.
5. A crash policy for unexpected update defects.
6. Optional time-budget yielding for message bursts.

Foldkit's five-millisecond yield is useful as a fairness mechanism, but it is
not a priority scheduler. Effract should not present it as equivalent to React
lanes.

### 7.2 Synchronous update, asynchronous commands

`update` should be synchronous and deterministic. Commands should start after
the current update stack, ideally from a microtask or Effect runtime boundary.
This ensures a command result cannot re-enter the current update recursively.

```text
dispatch(A)
  -> update(A)
  -> commit Model A
  -> enqueue/start Commands after update returns
  -> later Command result dispatches B
```

Multiple commands returned by one update should be concurrent by default only if
the feature declares that order is irrelevant. Otherwise the API should make
serial or keyed behavior explicit.

### 7.3 Stable snapshots

React's `useSyncExternalStore` requires that `getSnapshot` return a coherent,
cached value. The runtime should:

- publish one snapshot object per committed state version;
- preserve object identity when no relevant state changed;
- never expose a half-applied multi-store transition;
- notify React only after the snapshot is complete;
- keep server snapshots explicit for SSR and hydration.

If multiple atoms participate in one feature transition, publish one combined
feature snapshot rather than notifying React after every individual write.

---

## 8. Reactivity and Derived State

### 8.1 Push, pull, and push-pull

There are three useful strategies.

#### Push

Writes immediately notify every dependent node.

```text
write -> traverse dependents -> recompute -> notify
```

This has low read cost but can do unnecessary work during bursts.

#### Pull

Reads recursively evaluate current source values.

```text
read -> recursively read sources -> compute result
```

This avoids maintaining a large cached graph but may repeat computation.

#### Push-pull

Writes mark nodes dirty. A later read or scheduled flush recomputes only dirty
nodes.

```text
write -> mark dirty
later read/flush -> recompute dirty graph -> compare -> publish
```

Livestore uses a graph with dirty thunks and eager effect refresh. Unitflow uses
explicit source graphs with pull-based derivation and stream watchers. Effract's
current `Computation` is closer to eager push propagation.

### 8.2 Recommended hybrid

Use different policies for different node types:

- **Canonical input state:** update immediately.
- **Cheap derived values:** recompute synchronously.
- **Expensive derived values:** mark dirty and schedule.
- **Async queries:** invalidate and refresh through Effect fibers.
- **React notification:** publish stable snapshots after the chosen flush.

This preserves `useSyncExternalStore` correctness because the canonical state is
always current, while allowing expensive derivations and background refreshes to
be scheduled.

### 8.3 Dependency tracking choices

Effract's traced selectors are powerful:

```ts
const filtered = observe(($) => {
  const todos = $(todosAtom)
  const filter = $(filterAtom)
  return filterTodos(todos, filter)
})
```

Dynamic control flow can change dependencies naturally. The cost is that the
selector must execute to discover dependencies, and the runtime must carefully
replace old subscriptions.

Explicit dependencies are more predictable:

```ts
const filtered = combine(
  [todosAtom, filterAtom],
  (todos, filter) => filterTodos(todos, filter),
)
```

The recommended design supports both:

- traced dependencies for ergonomic local derivations;
- explicit dependencies for stable feature graphs, database queries, and
  resource lifetimes.

### 8.4 Equality

Equality should be a per-node policy rather than one universal rule.

- `Object.is` is cheap and appropriate for immutable references.
- structural equality helps for small configuration values and query keys;
- schema equivalence is useful for typed query results;
- always-change invalidation is necessary for table/version tokens;
- custom equality is useful for large derived view models.

Deep equality on every keystroke is usually wasteful. Conversely, query keys and
configuration records can benefit from structural equality.

### 8.5 Avoid duplicated derived state

Persist canonical state and derive views where practical:

```text
canonical todos + filter
  -> visible todos
  -> count
  -> empty-state flag
```

Duplicating `visibleTodos`, `remainingCount`, and `hasResults` as writable state
creates synchronization paths and replay problems. A durable read model is a
different case: it is a deliberate projection with its own rebuild and version
semantics.

### 8.6 Cycles

Reactive graphs and message handlers can create cycles:

```text
A changes -> B changes -> A changes
```

The runtime should detect or bound cycles at the message/runtime level. Useful
protections include:

- maximum cascading messages per drain;
- causality IDs on messages;
- detection of repeated `(node, version)` transitions;
- explicit cycle diagnostics;
- cancellation of runaway Effect fibers.

Silently allowing a productive cycle to run forever also prevents deterministic
quiescence tests from completing.

---

## 9. Purity and Effect Ownership

### 9.1 Purity boundary

The desired boundary is:

```text
Pure functions:
  update
  reducers/materializers
  derived selectors
  view functions

Effectful programs:
  commands
  subscriptions
  resource acquisition
  persistence
  network and browser integration
```

TypeScript cannot enforce all of this. The library must use API design,
conventions, runtime checks where useful, and tests.

### 9.2 What must not happen in update

`update` should not:

- call `Effect.runSync` or `Effect.runPromise`;
- start a fiber;
- read the wall clock or random source directly;
- mutate an atom behind the runtime's back;
- mutate the incoming Model;
- call a network or browser API;
- depend on mutable singleton state;
- perform logging that changes behavior.

Instead, describe those operations as Commands or return data needed by a
Command.

### 9.3 Materializers and reducers

For event-sourced state, materializers should be pure and deterministic:

```ts
const applyEvent = (state: State, event: DomainEvent): State => {
  // No I/O, no clock, no random values, no external service calls.
  return matchEvent(state, event)
}
```

If an event handler needs external data, capture the external result in the
accepted event or in a separately recorded interaction event. Otherwise replay
can produce a different state.

### 9.4 Effects must be replay-safe

During projection rebuild or DevTools replay:

- network sends must be disabled;
- commands must not execute;
- browser mutations must not occur;
- external gateways must be replay-aware;
- nondeterministic inputs must be recorded or injected.

This is why Foldkit's replay model correctly avoids executing Commands.

---

## 10. Managed Resources and Lifecycle

Foldkit's ManagedResources are one of the most valuable ideas to carry over.
They make resource requirements declarative and keep ownership in Effect rather
than React.

```ts
const SocketResource = ManagedResource.make({
  requirements: Schema.Struct({ roomId: Schema.String }),
  acquire: ({ roomId }) => ChatSocket.connect(roomId),
  release: (socket) => socket.close(),
})
```

Lifecycle should look like:

```text
None -> requirements A
  acquire A
  publish Ready(A)

requirements A -> requirements B
  stop subscriptions using A
  release A
  acquire B
  publish Ready(B)

requirements -> None
  interrupt fibers
  release resource
  publish unavailable
```

Recommended ownership hierarchy:

```text
Runtime scope
  -> feature scope
    -> model/query scope
      -> component scope where truly necessary
```

React render should not acquire long-lived resources. Subscription and committed
runtime ownership should control acquisition so abandoned renders do not leak
fibers or sockets.

---

## 11. Feature Composition

Foldkit's Submodels suggest a useful Effract feature abstraction.

```ts
interface Feature<Model, Message> {
  readonly initial: Model
  readonly update: Update<Model, Message>
  readonly subscriptions?: Subscriptions<Model, Message>
  readonly resources?: Resources<Model, Message>
  readonly view: React.ComponentType<FeatureViewProps<Model, Message>>
}
```

A parent feature should compose a child through stable identity and message
lifting:

```ts
type ParentMessage =
  | { _tag: "ParentOpened" }
  | { _tag: "Child"; message: ChildMessage }

const childMessage = (message: ChildMessage): ParentMessage => ({
  _tag: "Child",
  message,
})
```

Stable identity matters for:

- child state;
- Effect scopes;
- subscriptions;
- query cache entries;
- resource reuse;
- DevTools paths;
- dynamic lists.

For dynamic children, publish the new collection composition before closing
removed child scopes. This lets derived selectors stop observing removed children
before their channels are destroyed.

---

## 12. React Integration

The recommended React bridge is deliberately thinner than Foldkit's renderer.

```text
Effect feature runtime
  - owns Model
  - owns Message queue
  - owns Commands
  - owns Subscriptions
  - owns resources
  - exposes snapshot and subscribe

React adapter
  - useSyncExternalStore
  - dispatches Messages from event handlers
  - renders snapshot
  - uses startTransition for appropriate notifications
```

RECs can remain useful as a declarative way to read atoms, queries, and child
placements. However, long-lived Effects should not be interpreted as a side
effect of a speculative React render.

### 12.1 Why not run the whole Elm runtime during render?

React may start, interrupt, abandon, and restart renders. If rendering runs
message processing or starts Effect fibers, abandoned work can leak state changes
or resources.

React render must therefore read committed snapshots. Runtime transitions should
originate from:

- event handlers;
- external subscriptions;
- command completion;
- explicit runtime dispatch;
- committed lifecycle effects.

### 12.2 `startTransition`

`startTransition` should not retroactively reprioritize domain work that already
started. A useful design is:

```ts
dispatch(message, { priority: "urgent" })
dispatch(message, { priority: "background" })
```

The runtime may schedule recomputation or React notification differently, but it
should preserve committed-state correctness. Canonical writes should generally
be immediate; expensive derived work and renderer notification may be deferred
where safe.

Do not delay the store mutation itself in a way that makes `getSnapshot` return a
value inconsistent with the latest committed external state.

---

## 13. Async Queries and Mutations

Async state should be represented as ordinary runtime state where possible:

```ts
  | { _tag: "Initial" }
  | { _tag: "Loading"; previous?: A }
  | { _tag: "Success"; value: A }
  | { _tag: "Failure"; error: E; previous?: A }
```

This allows loading, previous data, failure, optimistic updates, persistence,
and React rendering to use the same dataflow machinery.

Effract's existing query cache remains valuable for request deduplication,
Suspense, stale-while-revalidate, and runtime-scoped lifetimes. It should be
connected to the feature runtime rather than bypassing it.

Every async request that can overlap should have a correctness policy:

```text
request generation 1 starts
request generation 2 starts
generation 2 completes -> publish
generation 1 completes -> ignore as stale
```

Effect interruption is useful but not always sufficient. A request may already
have completed or an adapter may not support immediate cancellation. Generation
tokens or request identities provide a second line of defense.

---

## 14. Event-Sourced Domain Integration

Livestore's architecture suggests that Effract should support a lower-level
domain event adapter rather than making every atom durable.

```text
React intent
  -> Feature Message
  -> Domain Command
  -> Event store commit
  -> materializer applies events
  -> table/entity version invalidation
  -> query recomputation
  -> React snapshot
```

### 14.1 Invalidation granularity

For database-backed state, a derived query should depend on invalidation tokens
such as:

- table version;
- entity version;
- aggregate version;
- query dependency set.

It should not need to subscribe to every individual row object. This is the key
lesson from Livestore's table references.

### 14.2 Optimistic updates

For collaborative or offline state:

1. Commit the local event optimistically.
2. Materialize it into local read state.
3. Queue it for synchronization.
4. Receive acceptance, rejection, or rebase information.
5. Roll back/reapply as required.
6. Publish one coherent new snapshot.

This is a domain concern and should not be confused with React transitions.

### 14.3 Read models

The React layer should consume read models shaped for rendering:

```ts
  readonly items: ReadonlyArray<{
    readonly id: string
    readonly title: string
    readonly completed: boolean
  }>
  readonly remaining: number
}
```

The write model may use aggregates and invariants that are entirely different.
The read model can be cached, denormalized, indexed, or rebuilt independently.

---

## 15. Scheduler Design

Foldkit exposes three useful timing domains:

```text
update       = synchronous and ordered
command      = microtask-started Effect fiber
render       = renderer-controlled and coalesced
```

Effract should make these boundaries explicit.

### 15.1 Proposed priority classes

```ts
  | "urgent"
  | "normal"
  | "background"
  | "idle"
```

Possible policies:

- urgent messages drain immediately;
- normal messages drain in FIFO batches;
- background refreshes use Effect schedules and can yield;
- idle work waits for no urgent messages;
- derived computations can declare cost and priority;
- React notification can be wrapped in `startTransition` for non-urgent work.

The runtime should not attempt to reimplement React's lane scheduler. It should
provide dataflow policy and let React schedule rendering.

### 15.2 Batching

Batching should be explicit at the snapshot boundary:

```text
Message A -> Model 1
Message B -> Model 2
Message C -> Model 3
                    |
                    +-> publish Model 3 once
```

Whether intermediate Models are externally observable should be a documented
contract. Effect subscribers may need to observe every transition while React
only needs the final snapshot. If both are required, expose separate APIs.

### 15.3 Quiescence

Tests need a deterministic way to wait for an Effect cascade:

```ts
  runtime.dispatch(Requested())
)
```

This should wait for counted message, command, and subscription work. It must
not claim that React has committed a render. A separate test helper can await a
React commit when required.

---

## 16. Testing and DevTools

Adopt two primary test layers.

### Story tests

Story tests execute the feature without React:

```ts
const story = Story.with(initialModel)

story
  .send(LoadRequested())
  .expectModel((model) => model.loading)
  .resolve(LoadTodos, TodosLoaded({ todos }))
  .expectModel((model) => model.todos.length === 1)
```

They should verify:

- state transitions;
- returned commands;
- command ordering policies;
- failure handling;
- subscription changes;
- cancellation and stale result behavior.

### Scene tests

Scene tests use the real React binding or a headless renderer:

```ts
const scene = Scene.start(feature)

scene.getByRole("button", { name: "Save" }).click()
scene.expectText("Saved")
```

They should verify the actual event-to-message-to-render path, accessibility,
and resource cleanup.

### Replay

Replay should:

- restore a known state or snapshot;
- apply messages/events deterministically;
- never execute Commands;
- never send external notifications;
- expose causality and state diffs;
- distinguish omitted history from unchanged state.

Durable event replay and DevTools message replay should be separate APIs.

---

## 17. Recommended Public API Shape

A possible feature API is:

```ts
const Todos = Feature.make({
  initial: {
    items: [],
    loading: false,
  },

  update: (model, message) => {
    switch (message._tag) {
      case "LoadRequested":
        return [
          { ...model, loading: true },
          [
            TodosApi.list.pipe(
              Effect.map((items) => ({ _tag: "Loaded", items })),
              Effect.catchAll((error) =>
                Effect.succeed({ _tag: "LoadFailed", error }),
              ),
            ),
          ],
        ]

      case "Loaded":
        return [{ items: message.items, loading: false }, []]
    }
  },

  subscriptions: [todosRefreshSubscription],
  resources: [todosSocketResource],
})
```

The React side should look conceptually like:

```tsx
function TodosView() {
  const { snapshot, dispatch } = useFeature(Todos)

  return (
    <>
      <button onClick={() => dispatch({ _tag: "LoadRequested" })}>
        Load
      </button>
      {snapshot.loading ? "Loading..." : snapshot.items.length}
    </>
  )
}
```

The exact API can differ, but the capability boundary should remain:

- views read snapshots;
- event handlers dispatch intents/messages;
- only the runtime executes commands and owns resources;
- domain writes are not arbitrary atom mutations.

---

## Architectural Rationale

### Foldkit's biggest contribution: one explicit transition path

The important Foldkit feature is not merely Commands, Subscriptions, or a
different renderer. It is the invariant:

```text
The only way canonical Model state changes is through update(Model, Message).
```

Effract currently has a powerful reactive storage layer, but canonical state can
be changed through arbitrary atom writes:

```text
service -> atom.set
component -> atom.update
query callback -> atom.set
WebSocket callback -> atom.update
```

That makes the current value observable but not necessarily explainable. The
Foldkit-style runtime changes the programming model:

```text
React event / transport / subscription
  -> typed Event
  -> one ordered transition function
  -> State + Commands
```

This provides a stronger developer experience:

- State changes have a discoverable cause.
- Tagged event unions make the transition vocabulary explicit.
- Exhaustive matching makes missing transition logic visible to TypeScript.
- Commands are visible outputs of a transition rather than hidden writes.
- Async results re-enter through the same event path.
- Story tests can run the feature without React.
- Debugging can show event, previous state, next state, projections, and commands.
- Replay can reproduce state when the event inputs are deterministic.
- Feature behavior is easier to document because the state machine is explicit.

This is the mental shift that turns Effract from a collection of reactive cells
into an application architecture.

### What must be sacrificed

The transition model has real costs:

- Canonical domain state cannot have unrestricted public setters.
- Features need explicit Event, State, and Command types.
- Async workflows become visible state machines instead of hidden callbacks.
- Simple local mutations require more ceremony.
- State updates must be immutable or otherwise transactionally controlled.
- A runtime, queue, root Scope, and lifecycle protocol must be maintained.
- Event schemas and sequence semantics become part of the application contract.

These costs should be paid for feature and domain state, not necessarily for
every local UI detail. A draft text value or hover state can still use an atom
when it has no meaningful application transition semantics.

The intended rule is:

```text
Canonical feature/domain state -> Event runtime
Derived state                -> Effract graph
Ephemeral local interaction  -> atom when appropriate
```

### Centralized transitions do not mean broad React updates

The root runtime can own one canonical `AppState` while Effract exposes
fine-grained projections:

```text
AppState
  -> visibleTodos projection
  -> selectedTodo projection
  -> unreadCount projection
```

A component subscribes to its projection through `useSyncExternalStore`, not to
the whole application Model. The result preserves Effract's graph-like
component-level reactivity and avoids React Context-style parent/child update
coupling.

### Why defer rendering to React?

Foldkit owns Snabbdom, so it can control the whole pipeline:

```text
Message -> synchronous update -> rAF render -> DOM patch
```

Effract should not reproduce that renderer-owned flow. React can interrupt,
restart, defer, merge, and abandon rendering. The frontend runtime should
commit state independently and notify React only after a coherent snapshot is
available:

```text
Event -> State commit -> projection update -> React notification
                                      -> React scheduling -> commit
```

This means React may skip intermediate visual states while the event runtime
still processes every event in order. That is an advantage for input
responsiveness and expensive rendering. It also means the library must document
that "state committed" and "React DOM committed" are different moments.

View RECs therefore become readers of projected state. Runtime Effects,
commands, subscriptions, and resource acquisition run in the frontend Effect
Scope outside speculative React render. A render REC may read stable snapshots
and restart-safe Suspense queries, but it must not perform externally observable
commands.

### Atomic Event -> State commit

The frontend needs a database-like atomicity guarantee even when its state is
in memory:

```text
No observer sees the event without its state,
or the state without its event.
```

Use one serialized event processor and a synchronous transition function. Effect
`Queue` can provide ingress, but one processor must be the only writer of
canonical state:

```text
all event sources
  -> FIFO event queue
  -> one transition processor
  -> synchronous update/materializer
```

One commit should perform these steps without awaiting:

1. Dequeue the Event.
2. Assign a monotonic frontend sequence number.
3. Read the current State.
4. Compute the next State synchronously.
5. Validate the transition.
6. Record Event metadata and causality.
7. Swap canonical State exactly once.
8. Increment the State/snapshot version.
9. Invalidate or recompute affected projections.
10. Publish one complete snapshot notification.
11. Start returned Commands after the commit.

The logical commit is:

```ts
type Commit<State, Event> = {
  readonly sequence: number
  readonly event: Event
  readonly previousState: State
  readonly nextState: State
}
```

If the transition fails, State remains unchanged and Commands do not start. If a
Command fails later, it dispatches a failure Event; it does not mutate State
directly.

Messages arriving during a transition are queued rather than recursively
processed:

```text
queue = [A, B]
process A, which dispatches C
next queue = [B, C]
```

This preserves order and prevents a command or subscription from starving work
that was already queued.

### Ordering across asynchronous sources

The runtime can guarantee local queue order, but HTTP responses and WebSocket
messages do not automatically have one global order. Events from the backend
should carry:

- server sequence or cursor;
- aggregate/version number;
- event ID;
- correlation ID;
- idempotency key where needed.

The frontend should reject, buffer, or recover from gaps rather than assuming
arrival order is authoritative. Concurrent commands should have explicit
policies: serial, keyed, latest-wins, or concurrent.

### Durable atomicity

In-memory State and an in-memory event history are atomic only for the current
runtime lifetime. If the browser needs durable offline state, the event append,
state snapshot, and outbox record need an IndexedDB transaction:

```text
IndexedDB transaction
  -> append local Event
  -> materialize State snapshot
  -> append synchronization outbox entry
  -> commit all together
```

For backend-authoritative state, the durable transaction belongs to the backend:

```text
backend database transaction
  -> validate Command
  -> append domain Event
  -> materialize backend State
  -> commit
```

The frontend then applies server-sequenced events or projections and reconciles
any optimistic local Events. This is where Livestore's sequencing,
materialization, invalidation, optimistic synchronization, and rebase patterns
become relevant.

### Livestore's specific lesson

Livestore does not simply write an Event and then notify React. Its commit path
encodes/sequences events, materializes them, updates synchronization state, and
only then refreshes affected table references. Effract should adopt the same
ordering principle:

```text
commit event/state first
  -> invalidate projections second
  -> notify renderer last
```

For in-memory Effract, the materializer is a pure state transition. For a
database-backed feature, the materializer may update an embedded store and
publish table/entity version tokens. In both cases, React sees only the
post-commit projection.

### Atomicity does not require React to render every state

The runtime may process:

```text
Event A -> State 1
Event B -> State 2
Event C -> State 3
```

React may render only State 3. That does not violate atomicity. Atomicity means
each individual transition is complete and ordered, not that React must commit a
DOM frame for every transition.

## 18. Adoption Plan

### Adopt first

1. Typed Message definitions for feature workflows.
2. Pure update functions for those workflows.
3. Commands that return result Messages.
4. Model-driven Effect subscriptions.
5. Root and feature Effect scopes.
6. Stable combined snapshots through `useSyncExternalStore`.
7. Story tests and deterministic Effect quiescence.
8. Typed Ports for host integration.

### Adopt selectively

1. A central message runtime and atomic transition processor.
2. Keyed command cancellation.
3. Priority and background scheduling.
4. ManagedResources.
5. Replayable feature history.
6. Explicit read models.
7. Livestore-style table/entity invalidation.

### Do not copy directly

1. Foldkit's Snabbdom renderer assumptions.
2. A module-global Model shared by every runtime, request, or mount.
3. Uncached, externally observable Effect execution during React render.
4. Treating every UI Message as a durable event.
5. Treating DevTools history as event sourcing.
6. Rebuilding React's scheduler inside the dataflow runtime.
7. Deep equality for every atom and every keystroke.

---

## 19. Architectural Decision Questions

The following decisions should be resolved before implementing the full
abstraction:

1. Is the message runtime application-wide, feature-local, or both?
2. Are atoms writable by consumers, or only through feature inputs/commands?
3. Are command results delivered as typed Messages, typed Events, or both?
4. Are updates synchronous by default?
5. Which command policies are built in: concurrent, serial, latest, keyed?
6. Is a feature's Model one atomic React snapshot?
7. Can Effect subscribers observe intermediate states hidden from React?
8. How does `startTransition` affect notification without affecting domain order?
9. Which state is ephemeral UI state and which is durable domain state?
10. Which domain events require persistence and schema versioning?
11. Are query projections eagerly maintained or lazily recomputed?
12. How are cycles detected and reported?
13. What counts as runtime quiescence in tests?
14. How are stale async results prevented from publishing?
15. How are component, feature, runtime, and request scopes related?

---

## 20. Final Position

Foldkit's most valuable contribution is not its renderer. It is the explicit
separation of:

- messages from state;
- state transitions from Effects;
- subscriptions from commands;
- resources from views;
- feature composition from incidental component nesting;
- update testing from rendering testing.

Elm contributes a disciplined one-way dataflow model. CQRS contributes the
separation of intent/write processing from read projections. Event sourcing
contributes durable history and rebuildable state when the domain needs it.
Livestore contributes event materialization, invalidation tokens, reactive
queries, optimistic sync, and conflict handling.

Effract contributes the React-native external-store bridge, fine-grained traced
reactivity, Effect-backed storage, Suspense, query caching, and explicit runtime
scopes.

The best Effract architecture combines these strengths without collapsing them
into one abstraction:

```text
React / REC views
  -> stable snapshots and typed intents

Feature runtime
  -> Messages
  -> pure updates
  -> Commands
  -> Subscriptions
  -> ManagedResources

Reactive layer
  -> derived state
  -> invalidation
  -> equality policies
  -> snapshot publication

Domain layer, where justified
  -> Commands
  -> durable Events
  -> materializers
  -> CQRS read models
  -> synchronization and replay
```

This gives Effract a clearer causal model and stronger lifecycle guarantees
without sacrificing React concurrency, fine-grained UI reactivity, or Effect's
structured concurrency.

The more precise architecture is described below: centralize the frontend event
runtime and canonical state transitions, keep Effract's graph as the fine-grained
projection layer, and let React schedule and commit views that read projected
state.

---

## 21. Revised Architecture: Event Model, State Model, and React Views

The central conclusion is more specific than "avoid a global Model." Effract
should have one authoritative frontend runtime per application instance. That
runtime should own the event model and canonical state transitions. It should
not be a module-global singleton, and it should not require React to own the
transition loop.

```text
Frontend Effect runtime scope
  Event model / message queue
    -> update / materializer
    -> canonical state model
    -> Effract reactive graph
    -> projected state
    -> React useSyncExternalStore
    -> REC generators read projections
    -> React schedules and commits rendering
```

The backend is a separate Effect runtime and scope tree:

```text
Backend Effect runtime
  domain commands
    -> durable events / database writes
    -> backend projections
    -> HTTP or WebSocket transport
    -> frontend transport event
    -> frontend event model
```

Effect `Runtime` executes Effects. Effect `Scope` owns resource lifetimes and
finalizers. These are not shared across a network boundary. A frontend
WebSocket fiber and a backend connection fiber are separate executions with
separate scopes. HTTP and WebSocket messages connect the two runtimes.

### 21.1 The frontend root scope

The intended frontend "global" is an application-owned root scope:

```text
Application scope
  - frontend event loop
  - command fibers
  - HTTP/WebSocket clients
  - subscriptions
  - caches
  - feature scopes
  - projected state
```

This scope can last for the lifetime of a mounted application. It must still be
created per application instance, and usually per server request during SSR.
That is different from module-level state shared by every request and mount.

### 21.2 Centralized transitions with fine-grained reactivity

Centralized ownership does not imply that every component subscribes to one
large snapshot or rerenders on every transition. There are three different
granularities:

```text
Canonical state ownership
  -> one frontend runtime

Reactive graph granularity
  -> atoms, computations, query nodes, projections

React subscription granularity
  -> the specific projected value a component reads
```

The application can have one canonical state model:

```ts
type AppState = {
  readonly auth: AuthState
  readonly todos: TodosState
  readonly editor: EditorState
  readonly notifications: NotificationState
}
```

while exposing fine-grained projections:

```text
AppState
  -> visibleTodos projection
  -> selectedTodo projection
  -> unreadNotificationCount projection
```

The existing Effract `Computation` mechanism is well suited to these
projections. A component reading `visibleTodos` need not be notified when only
`auth` changes. This is different from putting the whole Model in React Context
and broadly invalidating consumers when the provider value changes.

The goal is therefore:

> Centralize state ownership and transition authority while retaining
> fine-grained graph invalidation and component-level subscriptions.

## 22. Event Model and State Model

The event model and state model should be explicit, separate concepts.

### 22.1 Event model

The frontend event model contains ordered inputs to the runtime:

```ts
type AppEvent =
  | { readonly _tag: "SearchChanged"; readonly value: string }
  | { readonly _tag: "TodosLoadRequested" }
  | { readonly _tag: "TodosLoaded"; readonly todos: ReadonlyArray<Todo> }
  | { readonly _tag: "TransportMessageReceived"; readonly value: unknown }
```

Events enter through React event handlers, HTTP command results, WebSocket
messages, Effect subscriptions, timers, schedules, host Ports, or backend
domain events. The runtime processes them outside React render:

```text
enqueue event
  -> dequeue event
  -> update(state, event)
  -> commit state transition
  -> launch returned commands
  -> update subscriptions/resources
```

### 22.2 State model

The state model is the current materialized result of processing events. React
does not process the event queue as its normal render input. It reads state
projections:

```text
event model
  -> canonical state model
  -> reactive projection
  -> React snapshot
```

This is the frontend analogue of the Event -> State separation found in
event-sourced systems and Livestore. It is not automatically durable event
sourcing. Frontend events may be transient and may disappear when the
application closes.

Keep these concepts distinct:

```text
Intent       what the user wants
Command      Effect program that attempts work
Domain event accepted fact after validation
State        materialized current view of facts
Projection   derived state optimized for a consumer
```

For example:

```text
User clicks Book
  -> BookingRequested intent
  -> SubmitBooking Effect command
  -> backend accepts command
  -> BookingCreated domain event
  -> frontend transport event
  -> frontend state transition
  -> projected booking view
  -> React
```

`SearchChanged` normally does not belong in a durable event log. `BookingCreated`
may. The frontend event model can contain both, but the persistence boundary
must distinguish them.

## 23. Foldkit, Effract, and Livestore

These designs share an event-to-state idea but place the boundaries in
different layers.

### 23.1 Foldkit

Foldkit has one application runtime that owns the Model, Message queue,
synchronous `update`, Commands, Subscriptions, ManagedResources, renderer, and
root Effect scope:

```text
Message -> update(Model, Message) -> Model + Commands
```

Its Model is in-memory materialized state. Its DevTools history is bounded
debugging history, not a durable event log. Because Foldkit owns the renderer,
it can synchronously process Messages and schedule its own rAF patch.

### 23.2 Livestore

Livestore extends the event/state split with durable domain mechanics:

```text
store.commit(events)
  -> encode and sequence events
  -> materialize events into SQLite
  -> push/synchronize events
  -> invalidate table references
  -> recompute reactive queries
  -> notify React
```

Its important separation is between immutable events as the write/sync
protocol, materialized database state, table-level invalidation references,
reactive query projections, and renderer subscriptions.

Livestore's `ReactiveGraph` uses Ref -> Thunk -> Effect relationships. A table
Ref is an invalidation source, a query Thunk derives a result, and a
subscription Effect notifies the renderer. This is a useful model for Effract:
canonical events/state can remain centralized while query and component reads
remain fine-grained.

### 23.3 Effract

Current Effract combines more concerns in the atom/REC layer:

```text
atom.set/update
  -> AtomRef mutation
  -> immediate listeners
  -> Computation recomputation
  -> React useSyncExternalStore
```

The reactive graph is fine-grained, but arbitrary atom writes do not necessarily
pass through a central event model. The REC interpreter also runs inside React
render and may resolve Effects there.

### 23.4 Required Effract change

Add a frontend event runtime above or beside the existing graph:

```text
Frontend event runtime
  -> canonical state AtomRef / state store
  -> Effract Computation graph
  -> projected state atoms
  -> React
```

The graph remains the read/projection mechanism. It should not be replaced by
one coarse React subscription.

### 23.5 Required Foldkit-style change

A Foldkit-style library integrating a database or durable event stream needs an
explicit distinction between transient UI Messages and durable domain Events.
It would need event schemas and versioning, durable sequence numbers,
materializers, idempotent processing, retry and duplicate handling, transport
synchronization, projection invalidation, and snapshot/rebuild strategies.

Foldkit already has the event-to-Model transition boundary. Livestore supplies
the durable event, materialization, invalidation, and sync machinery. Effract
can combine the two by using a Foldkit-style frontend event loop and an
Effract-style fine-grained projection graph.

## 24. RECs as Readers of Projected State

The recommended REC model is a declarative reader of state projected by the
frontend runtime:

```ts
function* TodoView() {
  const visibleTodos = yield* Todos.visibleTodos
  const isLoading = yield* Todos.isLoading

  return <TodoList todos={visibleTodos} isLoading={isLoading} />
}
```

The event runtime starts independently:

```text
mount application
  -> create frontend ManagedRuntime
  -> open application Scope
  -> start event processor fiber
  -> start subscriptions and resources
  -> mount React views
```

React event handlers dispatch into the event model:

```tsx
<button
  onClick={() => runtime.dispatch({
    _tag: "TodoAddedRequested",
    text: draft,
  })}
>
  Add
</button>
```

Foldkit must own rendering because it owns Snabbdom and the DOM patch lifecycle.
It can define:

```text
Message -> synchronous update -> rAF patch latest Model
```

Effract does not own React's renderer. React can interrupt, restart, defer, or
abandon work and can commit only the current tree. Effract should exploit this
rather than imitate it:

```text
Effect runtime owns truth, causality, and resource lifetime.
Reactive graph owns projections and invalidation.
React owns view scheduling and commit.
RECs describe how views read projections.
```

Deferring rendering to React is therefore an advantage, not a compromise. The
runtime commits state and projections, then React decides when and whether to
render them. This is more flexible than Foldkit's renderer-owned flow while
retaining Foldkit's explicit transition model.

## 25. Effect Execution and React Render

The rule is not that Effects can never be yielded by a REC. The rule is that
render-time Effects must be classified and restart-safe.

### Runtime Effects

Commands, mutations, event processors, subscriptions, WebSocket loops, resource
acquisition, persistence, and transport sends run outside render and dispatch
events into the frontend event model.

### Render-safe reads

View RECs may read stable atom snapshots, derived projections, cached query
results, Suspense-managed reads, and immutable services that do not perform
external work during access.

Effract's existing suspensable store is relevant here. It can execute an async
read once, store its Promise/fiber in a runtime-scoped cache, suspend with
`React.use`, and reuse the result after React retries rendering. That is a
specialized render-safe read protocol, not permission for arbitrary Effect
execution in render.

Mutations and commands remain outside render:

```text
React event or committed lifecycle
  -> dispatch event
  -> command fiber
  -> result event
```

## 26. Equality: What Foldkit Actually Does

Foldkit does not use deep equality for every Model update. Its primary Model
change check is reference equality:

```text
nextModel === previousModel
```

Returning the same Model reference suppresses rendering. Returning a new Model
reference schedules rendering. Commands returned with the transition still run
even when the Model reference is unchanged.

This assumes immutable updates and structural sharing:

```text
previous.auth === next.auth
previous.settings === next.settings
previous.todos !== next.todos
```

Foldkit uses different equality boundaries for different problems:

- Model transition: reference equality.
- Subscription requirements: Schema-derived equivalence.
- VDOM reconciliation: VNode identity and structural comparison.
- Lazy view optimization: function/argument identity.

The lesson for Effract is not "use deep equality." It is:

> Choose equality at the boundary where the runtime needs to decide whether
> work should continue.

Effract currently gates atom writes with Effect's structural `Equal.equals`.
That is useful for query keys and small configuration records, but should not be
the only policy for every atom. Recommended defaults are:

```text
Canonical state transitions:
  immutable replacement + reference/version checks

Derived projections:
  per-node Object.is, shallow, schema, or custom equivalence

Query keys:
  structural Hash/Equal

Invalidation tokens:
  always-change or monotonic version

Event occurrences:
  never suppressed as state equality
```

For example, a `RefreshRequested` event is meaningful even if the resulting
data is structurally identical. Event occurrence and state value must not be
conflated.

## 27. Implementation Blueprint

The smallest architecture supporting this design contains these layers:

```ts
interface EventRuntime<State, Event> {
  readonly dispatch: (event: Event) => void
  readonly getState: () => State
  readonly subscribe: (listener: () => void) => () => void
  readonly scope: Scope.Scope
}
```

Internally it owns the FIFO event queue, processing guard, current canonical
state, state version, command fibers, subscription fibers, root Effect scope,
and causality metadata.

The transition function is synchronous and deterministic:

```ts
  state: State,
  event: Event,
) => readonly [State, ReadonlyArray<Command>]
```

It returns Effect commands as values rather than running Effects. The canonical
state is exposed to the Effract graph as a source node. Computations derive
`visibleTodos`, `unreadCount`, and other projections. RECs subscribe to those
projections through the existing `useSyncExternalStore` bridge.

Transport Effects map responses into frontend events:

```text
HTTP/WebSocket response
  -> decode
  -> frontend event
  -> transition
  -> projected state
  -> React
```

The HTTP Effect belongs to the frontend runtime scope. The server's Effect
runtime processes the request in a different process, runtime, and scope.

## 28. Final Revised Position

The target architecture is:

```text
Foldkit transition architecture
  + Livestore Event -> State and projection ideas
  + Effract fine-grained reactive graph
  + Effect frontend scopes and fibers
  + React scheduling and commit model
```

Foldkit provides:

```text
Message -> update -> Model + Commands
```

Livestore provides:

```text
Domain Event -> materialized state -> invalidated reactive query
```

Effract provides:

```text
projected state -> Computation -> useSyncExternalStore -> React
```

The core invariant becomes:

> All meaningful frontend state transitions are processed by the frontend
> Effect runtime's event model. React and View RECs only read the resulting
> projected state during render.

This keeps Foldkit's strongest property, avoids React Context-style broad
invalidation, uses Livestore's event/materialization distinction where durable
state is required, and lets React provide a more capable rendering scheduler
than a library-owned renderer can provide.

### Additional Effect references

- Effect Runtime: https://effect.website/docs/runtime/
- Effect Scope: https://effect.website/docs/resource-management/scope/
- Effect Fibers: https://effect.website/docs/concurrency/fibers/
- Effect Layers: https://effect.website/docs/requirements-management/layers/

---

## Sources

- Elm Guide, The Elm Architecture:
  https://guide.elm-lang.org/architecture/
- Elm Guide, Commands and Subscriptions:
  https://guide.elm-lang.org/effects/
- Elm Guide, Ports:
  https://guide.elm-lang.org/interop/ports.html
- Elm Guide, Html.lazy:
  https://guide.elm-lang.org/optimization/lazy.html
- Martin Fowler, CQRS:
  https://martinfowler.com/bliki/CQRS.html
- Microsoft Azure Architecture Center, CQRS Pattern:
  https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs
- Martin Fowler, Event Sourcing:
  https://martinfowler.com/eaaDev/EventSourcing.html
- Local research, Effract reactivity:
  `knowledge/effract-reactivity.md`
- Local research, Livestore versus Effract:
  `knowledge/livestore_vs_effract.md`
- Local research, Foldkit architecture:
  `knowledge/foldkit_arch.md`
