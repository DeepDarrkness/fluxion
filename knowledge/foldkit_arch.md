# Foldkit Implementation Knowledge

> Source analysis of `foldkit/foldkit` at commit `17d121ad` (cloned to
> `/tmp/opencode/foldkit`). Foldkit is a TypeScript frontend framework built on
> Effect and organized around the Elm Architecture.
>
> **Goal of this document:** Capture Foldkit's runtime, dataflow, lifecycle,
> rendering, composition, testing, and host-integration patterns, with emphasis
> on ideas that could be reused in a React UI layer with Effect owning the
> application dataflow.

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Core Application Model](#2-core-application-model)
3. [Runtime Startup and Scope](#3-runtime-startup-and-scope)
4. [Synchronous Message Scheduler](#4-synchronous-message-scheduler)
5. [Commands and Effect Execution](#5-commands-and-effect-execution)
6. [Subscriptions](#6-subscriptions)
7. [Managed Resources](#7-managed-resources)
8. [Rendering and Snabbdom](#8-rendering-and-snabbdom)
9. [Mount Lifecycle](#9-mount-lifecycle)
10. [Submodels and Message Lifting](#10-submodels-and-message-lifting)
11. [Routing and Navigation](#11-routing-and-navigation)
12. [Embedding and Typed Ports](#12-embedding-and-typed-ports)
13. [DevTools and Time Travel](#13-devtools-and-time-travel)
14. [Testing Model](#14-testing-model)
15. [HMR and Package Architecture](#15-hmr-and-package-architecture)
16. [Foldkit Compared with React](#16-foldkit-compared-with-react)
17. [Reusable Ideas](#17-reusable-ideas)
18. [Tradeoffs and Open Questions](#18-tradeoffs-and-open-questions)
19. [Key Files Reference](#19-key-files-reference)

---

## 1. Architecture Overview

Foldkit replaces the usual component-local state model with one application
runtime. The application author defines a typed Model, typed Messages, a pure
update function, and a view:

```text
DOM event / Port / Subscription / Command
                 |
                 v
              Message
                 |
                 v
       update(Model, Message)
                 |
                 +------------------+
                 |                  |
                 v                  v
              Model             Commands
                 |                  |
                 v                  v
          scheduled render    Effect fibers
                                      |
                                      v
                                  Messages
```

The public counter example demonstrates the basic contract:

```typescript
const update = (
  model: Model,
  message: Message,
): readonly [Model, ReadonlyArray<Command.Command<Message>>] =>
  M.value(message).pipe(
    M.withReturnType<readonly [Model, ReadonlyArray<Command.Command<Message>>]>(),
    M.tagsExhaustive({
      ClickedDecrement: () => [{ count: model.count - 1 }, []],
      ClickedIncrement: () => [{ count: model.count + 1 }, []],
      ClickedReset: () => [{ count: 0 }, []],
    }),
  )
```

Source: `examples/counter/src/main.ts`; the public runtime contract is exposed
by `packages/foldkit/src/runtime/public.ts` and implemented in
`packages/foldkit/src/runtime/runtime.ts`.

The central architectural distinction is that Effect is not interpreted during
view rendering. Effect is used to own asynchronous work, streams, services,
scopes, and cleanup. `update` itself is called synchronously by the runtime and
returns Effect-backed Commands as data.

Foldkit is therefore closest to:

- Elm's `Model -> Message -> (Model, Cmd Message)` architecture.
- An Effect application runtime wrapped around a synchronous reducer.
- A framework-level state machine whose renderer is Snabbdom rather than React.

It is not an event-sourced database. DevTools history is bounded debugging
history, not a durable event log, and the Model is in-memory application state.

---

## 2. Core Application Model

### 2.1 Application, Element, and Embed

Foldkit has three useful runtime entry points:

| API | Responsibility |
|---|---|
| `Runtime.makeApplication` | Owns a document, `<head>` metadata, URL routing, and body rendering. |
| `Runtime.makeElement` | Renders a Foldkit `Html` tree into an existing container; does not own page metadata. |
| `Runtime.embed` | Starts a host-controlled runtime and exposes only typed Ports plus `dispose`. |

The distinction is implemented in `runtime.ts` around lines 607-614 and
2864-2868. `embed` is implemented around lines 3414-3493.

The runtime does not expose its Model as a mutable object to the host. An
embedded consumer communicates through Schema-validated Ports:

```typescript
const inbound = Port.inbound(S.Struct({ step: S.Number }))
const outbound = Port.outbound(S.Struct({ count: S.Number }))

const handle = Runtime.embed(application, {
  ports: { inbound, outbound },
})

handle.ports.inbound.send({ step: 2 })
handle.ports.outbound.subscribe(value => {
  console.log(value.count)
})

handle.dispose()
```

The exact public Port constructors are in `packages/foldkit/src/port/port.ts`
and `packages/foldkit/src/port/public.ts`; the embedding example is in
`examples/embedding/src/main.ts` and `examples/embedding/src/host.ts`.

### 2.2 State Transition Contract

The update function returns a new Model and zero or more Commands:

```typescript
type Update<Model, Message> = (
  model: Model,
  message: Message,
) => readonly [Model, ReadonlyArray<Command<Message>>]
```

The runtime uses **reference equality** to determine whether the Model changed.
Returning the same Model suppresses rendering and HMR preservation, but Commands
returned in the tuple still execute. This behavior is in
`runtime/runtime.ts:1880-1957`.

The intended purity boundary is convention rather than a TypeScript guarantee:
the author can technically perform side effects inside `update`, but the
architecture asks them to describe effects as Commands, Subscriptions, Mounts,
or ManagedResources.

---

## 3. Runtime Startup and Scope

### 3.1 One Root Effect Scope

`startWith` wraps the runtime in `Effect.scoped` (`runtime/runtime.ts:1361-1381`).
The scope owns:

- Command fibers.
- Subscription streams.
- ManagedResource lifecycle streams.
- Mount-related work.
- Static resource Layer finalizers.
- Runtime schedulers and HMR preservation fibers.
- Browser and Port resources registered through Effect acquisition.

The runtime ends with `Effect.never` (`runtime/runtime.ts:2686-2690`) so the
scope remains alive until interrupted. Disposal is consequently scope
interruption rather than a normal application completion path.

### 3.2 Startup Ordering

The implementation performs startup in this broad order (`runtime.ts:1383-1735`
and `2247-2684`):

1. Create the runtime scope.
2. Lazily prepare the configured static resources Layer.
3. Create Port channels and the HTML boundary registry.
4. Create ManagedResource refs and their Effect service Layer.
5. Resolve flags and HMR-preserved Model data.
6. Resolve the initial URL and install browser navigation listeners.
7. Install crash handling and DevTools state.
8. Render the initial view.
9. Install Subscriptions and ManagedResource lifecycle fibers.
10. Fork initial Commands.
11. Mark boot complete and drain Messages buffered during startup.

The initial render occurs before Subscriptions and ManagedResources are attached.
Mount effects can therefore start during the initial patch, but their emitted
Messages are buffered until boot completion.

Static resources are built lazily and cached for the runtime lifetime. The Layer
build is uninterruptible and cached deliberately so a disposal race cannot cache
an interrupted construction (`runtime.ts:1383-1407`).

### 3.3 Disposal Ordering

`embed.dispose` is idempotent and interrupts the runtime fiber
(`runtime.ts:3479-3492`). The runtime registers finalizers so that, during
teardown:

1. Runtime dispatch is marked disposed.
2. Long-lived fibers are interrupted.
3. The current VNode tree is patched to `null`.
4. Mount destroy hooks run and dispatch teardown Messages, which are dropped.
5. The original host container is restored and emptied.

This is a strong lifecycle invariant: resources, effects, event listeners, and
DOM ownership all have one parent scope.

---

## 4. Synchronous Message Scheduler

This is one of Foldkit's most important implementation choices and one of the
largest differences from React.

### 4.1 Queue and Reentrancy

Foldkit does not use an Effect `Queue` as its primary Message scheduler. It uses
plain JavaScript state in `runtime.ts:1588-1645` and drains it in
`runtime.ts:1999-2053`:

```typescript
const pendingMessages: Message[] = []
let isProcessingMessages = false

function enqueueMessage(message: Message): void {
  pendingMessages.push(message)
  if (isProcessingMessages || !isBootComplete || isRenderingFrame) return
  processPendingMessages()
}
```

The exact implementation has additional disposal, crash, and scheduling guards,
but the semantic model is this:

1. Append a Message to a FIFO array.
2. If already draining, leave it queued.
3. If the runtime is patching the DOM, leave it queued.
4. Otherwise drain synchronously.
5. Messages dispatched during `update` go behind the current batch.
6. Messages dispatched during VDOM hooks wait until patching completes.

This makes Message processing deterministic and prevents recursive `update` calls.

### 4.2 Batch Snapshot

The drain takes a batch snapshot. Messages that existed when a batch began retain
priority over Messages appended while processing that batch. This matters when a
Command, Subscription, or event handler dispatches synchronously:

```text
pending = [A, B]

process A
  update(A) dispatches C

next batch = [B, C]
```

The ordering tests are in `packages/foldkit/src/runtime/messageProcessing.test.ts`.

### 4.3 Five-Millisecond Yield

Foldkit protects the browser from an unbounded synchronous Message burst. The
drain tracks cumulative synchronous work and, after roughly 5 ms, schedules the
remaining queue through `MessageChannel` (`runtime.ts:1959-1987` and
`2706-2710`).

```text
Message A -> update -> Message B -> update -> ...
                         |
                         +-- after budget --> MessageChannel
                                               |
                                               v
                                        continue FIFO drain
```

This is not React's scheduler. It has one coarse policy: process immediately
until a time budget is exceeded, then yield. It does not assign priorities,
lanes, transitions, or interruptible render work.

### 4.4 Command Asynchrony

Commands are always started from a microtask (`runtime.ts:1833-1877`). This
prevents a Command's Effect from running inside the current `update` call stack.
Consequently, a later synchronous user Message already present in the queue
cannot be overtaken by a Command result.

The scheduler therefore has three distinct timing domains:

| Work | Timing |
|---|---|
| `update` | Synchronous, FIFO, JavaScript stack. |
| Command start/result | Microtask start; result re-enters Message queue. |
| View patch | Next `requestAnimationFrame`, coalesced. |

---

## 5. Commands and Effect Execution

Commands are named Effects that return Messages. Their definitions live in
`packages/foldkit/src/command/index.ts` and `command/public.ts`.

```typescript
const FetchTodos = Command.define(
  'FetchTodos',
  S.Struct({}),
  Effect.gen(function* () {
    const todos = yield* TodosApi.list()
    return TodosLoaded({ todos })
  }),
)
```

The actual API supports typed argument schemas and several transformations:

- `Command.define` creates a named callable definition.
- `Command.mapEffect` transforms the underlying Effect.
- `Command.mapMessage` and `mapMessages` lift child results into parent Messages.
- Interruptible Commands associate invocations with keys so later work can
  interrupt active work.

Relevant source: `packages/foldkit/src/command/index.ts:242-337` and
`packages/foldkit/src/command/interruptible/interruptible.ts`.

At runtime, a returned Command is forked into the runtime scope. Its environment
contains static resources, ManagedResource refs, Port channels, and the
interruptible-command registry. A successful result is dispatched as a Message;
a defect or unrecovered failure is sent to the crash handler.

Commands in one update run concurrently:

```typescript
const update = (model, message) => {
  switch (message._tag) {
    case 'PageOpened':
      return [model, [LoadUser(), LoadNotifications()]]
  }
}
```

There is no implicit sequencing between `LoadUser` and `LoadNotifications`.
Applications must model dependencies explicitly through Messages or compose the
Effect before returning one Command.

---

## 6. Subscriptions

A Subscription is a model-dependent external stream. Its implementation is in
`packages/foldkit/src/subscription/subscription.ts` and its runtime integration
is in `runtime/runtime.ts:2452-2555`.

Conceptually:

```typescript
const clock = Subscription.make({
  schema: S.Struct({ running: S.Boolean }),
  modelToDependencies: model => ({ running: model.running }),
  dependenciesToStream: ({ running }) =>
    running ? Stream.tick(Duration.seconds(1)).pipe(Stream.map(Tick)) : Stream.empty,
})
```

The runtime:

1. Computes initial dependencies.
2. Starts with those dependencies even though no initial Model event exists.
3. Recomputes dependencies for each Model publication.
4. Applies Schema-derived equivalence.
5. Uses `Stream.switchMap` to interrupt the old stream and start the new one.
6. Dispatches emitted values as Messages.

`keepAliveEquivalence` permits a stream to remain alive while reading current
dependencies through a thunk. This is useful for streams whose connection should
not be recreated for every parameter change.

Browser helpers such as `Subscription.fromEvent` use Effect acquisition and
release to remove listeners when the stream is interrupted
(`subscription/fromEvent.ts:104-176`).

Subscriptions are not equivalent to Livestore queries. A Livestore query is a
reactive value over materialized database state; a Foldkit Subscription is an
Effect Stream whose lifetime is controlled by a projection of the Model.

---

## 7. Managed Resources

ManagedResources turn Model-driven requirements into scoped Effect services.
Implementation: `packages/foldkit/src/managedResource/managedResource.ts`; runtime
lifecycle: `runtime/runtime.ts:2583-2637`.

```typescript
const WebSocketResource = ManagedResource.make({
  tag: WebSocketTag,
  requirements: S.Struct({ roomId: S.String }),
  acquire: ({ roomId }) => WebSocket.connect(roomId),
  release: socket => socket.close(),
  onAcquired: socket => SocketReady({ socket }),
  onReleased: () => SocketClosed(),
  onAcquireError: error => SocketFailed({ error }),
})
```

The lifecycle is:

```text
None -> Some(requirements)
  acquire
  store Some(value)
  dispatch onAcquired

Some(A) -> Some(B)
  release A
  run finalizers for A
  acquire B

Some -> None
  release
  clear service ref
  dispatch onReleased
```

The resource ref is stored as `Ref<Option<Value>>`. Commands access it through
the registered service tag; access while inactive fails with
`ResourceNotAvailable`. Acquisition failures become Messages and leave the
resource unavailable. Release failures are swallowed by the runtime lifecycle.

The distinction from static `resources` is important:

- Static resources are built once and shared for the whole runtime.
- ManagedResources are repeatedly acquired and released as Model requirements
  change.

---

## 8. Rendering and Snabbdom

### 8.1 View Construction

The view is a plain function of the Model. The HTML DSL is implemented in
`packages/foldkit/src/html/index.ts`, with public exports in `html/public.ts`.
The VDOM implementation is in `packages/foldkit/src/snabbdom/` and `vdom.ts`.

```typescript
const { div, button, Class, OnClick } = html<Message>()

const view = (model: Model): Html =>
  div(
    [Class('counter')],
    [
      div([], [model.count.toString()]),
      button([OnClick(ClickedIncrement())], ['+']),
    ],
  )
```

Every view executes inside an HTML runtime frame containing the dispatcher,
Effect context, boundary registry, and current Submodel boundary
(`runtime.ts:2088-2103` and `2358-2375`).

### 8.2 rAF Render Coalescing

A Model change does not immediately patch the DOM. `runtime.ts:2314-2421`
schedules one `requestAnimationFrame` while a frame is pending:

```text
Message A -> Model 1 --+
Message B -> Model 2   +--> one rAF --> view(Model 2) --> patch
Message C -> Model 3 --+
```

The frame:

1. Clears the scheduled flag.
2. Stops if disposed, crashed, or DevTools-paused.
3. Resets the synchronous drain budget.
4. Builds the view from the latest Model.
5. Patches the VNode tree.
6. Applies document metadata for applications.
7. Attributes Mount lifecycle events to the current DevTools entry.
8. Drains Messages emitted during patching.

This makes Foldkit's rendering “latest Model per frame,” not “one render per
Message.” The Model transitions still happen synchronously; only DOM work is
frame-batched.

### 8.3 Snabbdom Patch Ordering

`packages/foldkit/src/snabbdom/init.ts` follows Snabbdom-like hook ordering:

- `create` runs as a VNode is created.
- `insert` runs after the patch has inserted queued VNodes.
- `destroy` runs before removal.
- Removal can wait for remove hooks.
- `postpatch` runs after child reconciliation.

This explains why Mount effects start after insertion and why Messages emitted by
DOM lifecycle hooks are held until the patch is complete.

---

## 9. Mount Lifecycle

Mounts attach Effect work to concrete VNodes. They are defined in
`packages/foldkit/src/mount/index.ts` and connected to HTML attributes in
`html/index.ts:2043-2099`.

```typescript
const OnMapMount = Mount.define(
  'OnMapMount',
  Effect.gen(function* () {
    const element = yield* Mount.element
    yield* MapLibrary.attach(element)
    return MapReady()
  }),
)
```

The lifecycle is:

```text
VNode insert
  -> start Mount Stream in captured Effect context
  -> dispatch emitted Messages

VNode destroy
  -> interrupt Mount fiber
  -> run Effect finalizers
  -> notify MountTracker
```

Mount arguments are captured when the VNode is constructed and are not refreshed
on ordinary rerenders (`mount/index.ts:187-193`). Stable VNode keys are therefore
important when an external DOM library should remain attached.

Mount failures are logged locally rather than sent through the terminal global
crash path. This is intentionally different from Commands and Subscriptions.

---

## 10. Submodels and Message Lifting

Submodels provide nested feature boundaries without creating independent
runtimes. The implementation is in `packages/foldkit/src/html/submodel.ts` and
`submodel/submodel.ts`.

```typescript
const ChildMessage = S.Union(ChildOpened, ChildClosed)

const childView = defineView<ChildMessage>((model, inputs) =>
  button(
    [OnClick(ChildOpened())],
    ['Open'],
  ),
)

const parentView = (model: ParentModel) =>
  h.submodel(
    'child-slot',
    childView,
    model.child,
    message => GotChildMessage({ message }),
  )
```

`h.submodel`:

1. Composes a boundary ID from the parent boundary and stable `slotId`.
2. Registers a child-to-parent mapper.
3. Pushes a child boundary while rendering.
4. Wraps top-level callback inputs.
5. Removes the boundary when the VNode is genuinely destroyed.

Event dispatch walks ancestor boundaries and applies each mapper
(`html/boundary.ts:221-304`). `OnUnmount` eagerly captures the mapper chain so
cleanup still dispatches correctly even if the boundary deregisters during the
same destroy phase.

Commands and other effects are lifted independently:

```typescript
const parentCommand = childCommand.mapMessages(
  message => GotChildMessage({ message }),
)
```

Source: `command/index.ts:242-337`, `subscription/subscription.ts`, and
`managedResource/managedResource.ts`. A Submodel is therefore a coordinated
message-routing and lifecycle boundary, not just a reusable view component.

---

## 11. Routing and Navigation

Routes are bidirectional parser/printers in `packages/foldkit/src/route/parser.ts`.
They support literal segments, typed segments, Schema-backed values, rest
segments, query values, and alternatives.

```typescript
const userRoute = Route.make(
  Route.path('users', Route.int),
  ({ value }) => UserRoute({ id: value }),
)

const url = Route.build(userRoute, UserRoute({ id: 42 }))
const route = Route.parse(userRoute, url)
```

`mapTo` turns parsed records into tagged route constructors and typed builders
(`route/parser.ts:535-571`). `oneOf` requires complete path consumption and uses
first-match precedence (`route/parser.ts:503-532`), so broad `rest` routes must
come after specific routes.

Navigation translates browser URL requests into typed internal or external
values (`navigation/urlRequest.ts:6-23`). Runtime startup reads the initial URL
and installs listeners (`runtime.ts:1650-1689`). `Transition` provides entered,
exited, and stayed policies (`route/transition.ts:33-140`).

One limitation observed in the parser is that repeated query keys are reduced
into the final record value rather than represented as a first-class multi-value
query type.

---

## 12. Embedding and Typed Ports

Ports define the host/runtime boundary in `packages/foldkit/src/port/port.ts`:

- `Port.inbound(schema)` accepts values from the host.
- `Port.outbound(schema)` emits values to the host.
- `Port.stream(inbound)` exposes an inbound stream.
- `Port.subscription(inbound, toMessage)` turns inbound values into Messages.
- `Port.emit(outbound, value)` sends a typed outbound value.

```typescript
const SetTheme = Port.inbound(S.Literal('light', 'dark'))
const ThemeChanged = Port.outbound(S.Struct({ theme: S.String }))

const app = Runtime.makeApplication({
  ports: {
    setTheme: SetTheme,
    themeChanged: ThemeChanged,
  },
  // ...
})
```

Port semantics:

- Inbound values are Schema-decoded before entering the application.
- Invalid values are rejected at the boundary.
- Values sent before boot are buffered in order.
- Values sent after an inbound stream is detached are dropped.
- Outbound delivery is microtask-deferred to prevent host reentrancy.
- Duplicate names across directions and duplicate Port objects are rejected.

The host receives only an `EmbedHandle` containing Ports and `dispose`
(`runtime.ts:980-1043`). This is a useful design for React integration: React
can own the host element and lifecycle without becoming the source of truth for
the application Model.

---

## 13. DevTools and Time Travel

Runtime-side DevTools code is in `packages/foldkit/src/devTools/`; the browser
overlay is in `packages/devtools/src/`.

History entries include:

- Message.
- Previous and next Model.
- Commands returned by update.
- Whether the Model changed.
- Model diffs.
- Mount starts and ends.
- Timestamps and runtime state.

The store uses bounded history and keyframes (`devTools/store.ts:134-165` and
`216-250`). Historical Models are reconstructed by replaying `update` from the
nearest keyframe. Replay does not execute Commands. Rendering uses a no-op
dispatcher so Mount Effects can establish DOM behavior without mutating live
history (`runtime.ts:2200-2229`).

Excluded Messages force a keyframe interval of one because replay cannot infer
state changes from omitted Messages (`runtime.ts:2187-2244`).

The DevTools MCP integration exposes Model, history, Message schemas, and
validated Message dispatch through `packages/devtools-mcp/src/tools.ts` and
`server.ts`. This is an interesting extension of Schema-driven application
boundaries for AI tooling.

---

## 14. Testing Model

Foldkit has two complementary testing layers.

### 14.1 Story Tests

`packages/foldkit/src/test/story.ts` executes `update` directly. Commands are
recorded as pending values and must be resolved explicitly:

```typescript
const story = Story.with(initialModel)

const next = story.send(ClickedLoad())

next
  .expectModel(model => model.loading === true)
  .resolve(LoadTodos, TodosLoaded({ todos }))
  .expectModel(model => model.todos.length === 1)
```

Story tests can assert Models, exact or partial Commands, OutMessages, and
cascading command resolution. `resolveAll` has a maximum cascade depth of 100
(`test/internal.ts:263-338`).

### 14.2 Scene Tests

`packages/foldkit/src/test/scene.ts` renders the actual VNode tree and simulates
accessible interactions without a browser:

```typescript
const scene = Scene.start({ model, update, view })

scene.getByRole('button', { name: 'Increment' }).click()
scene.expectText('1')
```

Scene supports roles, labels, text, selectors, test IDs, keyboard/pointer/input
events, scoped queries, and Mount lifecycle assertions. It explicitly tracks
unacknowledged Mount cleanup (`scene.ts:282-343` and `757-920`).

The separation is valuable:

- Story tests verify the dataflow contract without rendering or running Effects.
- Scene tests verify the real view and event-to-Message path deterministically.

---

## 15. HMR and Package Architecture

HMR preserves the Model across a Vite full reload rather than performing
component-level hot replacement. The Model is Schema-encoded and stored by a
runtime ID. Preservation is debounced by 200 ms and flushed on
`vite:beforeFullReload` (`runtime.ts:1523-1569` and
`runtime/preserveScheduler.ts:31-82`).

Runtime IDs are derived from `container.id`; missing IDs fail startup
(`runtime.ts:1355-1373`). This is useful for persistence identity but imposes a
constraint on multiple runtimes and reused host markup.

The core package is split into explicit subpath exports (`packages/foldkit/package.json`):

```text
foldkit/runtime
foldkit/command
foldkit/subscription
foldkit/managedResource
foldkit/mount
foldkit/port
foldkit/route
foldkit/story
foldkit/scene
foldkit/html
```

The repository also contains `packages/ui`, `packages/devtools`,
`packages/devtools-mcp`, `packages/vite-plugin-foldkit`, and an app generator.
The UI package uses stateless render helpers and stateful Submodels. It declares
`foldkit` and `effect` as peer dependencies to avoid duplicate Schema and runtime
identities (`packages/ui/package.json:124-140`).

---

## 16. Foldkit Compared with React

### 16.1 State and Render Boundaries

React treats rendering as the primary execution model. A state update schedules
work in React's scheduler, React may start, pause, abandon, restart, or commit a
render, and external stores must satisfy `useSyncExternalStore` snapshot rules.

Foldkit separates state transition from rendering:

```text
Foldkit:
  Message -> synchronous update -> Model mutation in runtime
          -> rAF render of latest Model

React:
  event/store update -> React scheduler -> render work
                      -> commit if work is still current
```

Foldkit's `update` is never a speculative React render. It is a committed
state-machine transition as soon as it runs. The subsequent DOM render can be
coalesced, but the Model transition is not paused or discarded.

### 16.2 Scheduler Differences

| Concern | Foldkit | React |
|---|---|---|
| State transition | Synchronous FIFO Message drain | Scheduled state update / render work |
| Priority | None; one coarse 5 ms yield policy | Lanes, priorities, transitions, cooperative scheduling |
| Reentrancy | Explicit pending array and batch snapshots | Updates enter React's scheduler and may be merged |
| Rendering | One rAF patch of latest Model | Scheduler-controlled render and commit |
| Render interruption | Not during a view build/patch | Render work can be interrupted and restarted |
| External store contract | Not needed for its own runtime | `useSyncExternalStore` requires committed snapshots |
| Async work | Effect fibers and Commands | Usually Effects, caches, actions, or external libraries |
| Cleanup | Root Effect scope interruption | Component/provider unmount and Effect cleanup |

Foldkit's 5 ms yield is a cooperative fairness mechanism, not a priority
scheduler. A burst of input, a websocket stream, and a background refresh all
enter the same FIFO queue. React can distinguish urgent input from transition or
background work; Foldkit does not unless the application models such policies
explicitly.

### 16.3 Why Foldkit Can Be Simpler

Foldkit owns the full renderer and runtime. It can safely assume:

- The Model is read by one runtime.
- Messages are processed in one known order.
- DOM patching is controlled by the runtime.
- A view is a pure function called at the selected frame boundary.

A React library cannot make those assumptions if it exposes an external store.
The store must publish a coherent snapshot to concurrent React renders, avoid
tearing, and integrate with transitions. A React version should likely expose a
`useSyncExternalStore` adapter while keeping Effect scheduling and dataflow
outside React.

### 16.4 Why React Can Be More Responsive

Foldkit's update path is synchronous until its time budget is exceeded. A large
single `update` or a chain of updates can delay input handling. Rendering is
frame-batched, but Model computation is not priority-aware.

React can defer non-urgent rendering and abandon work that is no longer useful.
That flexibility introduces constraints Foldkit does not have: render functions
must be pure, snapshots must be stable, and side effects must wait for commit.

### 16.5 React as a Thin View Layer

The most compatible design is not to port Foldkit's renderer wholesale. Instead:

```text
Effect runtime / Store
  - owns Model
  - owns Message queue
  - owns Commands, Streams, Resources
  - exposes snapshot + subscribe

React adapter
  - useSyncExternalStore(subscribe, getSnapshot)
  - renders snapshot
  - dispatches Messages from event handlers
  - uses transitions for non-urgent invalidations where appropriate
```

The React adapter should not run Effect programs during render. It should read a
committed snapshot and let the Effect runtime run Commands and resource
lifecycle work independently.

---

## 17. Reusable Ideas

This section lists implementation ideas worth carrying into a React-plus-Effect
library. Each item includes a concrete source reference and a minimal example.

### 17.1 One Root Scope for All Long-Lived Work

**Source:** `packages/foldkit/src/runtime/runtime.ts:1361-1407`,
`2649-2690`.

```typescript
const program = Effect.scoped(
  Effect.gen(function* () {
    const runtime = yield* makeRuntime
    yield* Effect.never
  }),
)
```

Own Commands, subscriptions, resource Layers, and browser resources under one
scope. React unmount can interrupt that scope and guarantee that no old fiber
can publish into a new component instance.

### 17.2 Commands as Typed Effect-to-Message Values

**Source:** `packages/foldkit/src/command/index.ts:242-337`,
`runtime/runtime.ts:1833-1957`.

```typescript
const Load = Command.define(
  'Load',
  S.Struct({ id: S.String }),
  Effect.gen(function* () {
    const result = yield* Api.load
    return Loaded({ result })
  }),
)

const [nextModel, commands] = update(model, Requested({ id: '1' }))
```

Keep Effect execution out of React render. Return typed Effects that re-enter the
central Message channel.

### 17.3 Model-Driven Stream Switching

**Source:** `packages/foldkit/src/subscription/subscription.ts`,
`runtime/runtime.ts:2452-2555`.

```typescript
const subscription = Subscription.make({
  schema: S.Struct({ room: S.Option(S.String) }),
  modelToDependencies: model => ({ room: model.room }),
  dependenciesToStream: ({ room }) =>
    Option.match(room, {
      onNone: () => Stream.empty,
      onSome: id => Chat.events(id),
    }),
})
```

Use `switchMap` semantics so changing a Model requirement interrupts the old
stream before the replacement owns the same logical resource.

### 17.4 Model-Driven Managed Resources

**Source:** `packages/foldkit/src/managedResource/managedResource.ts:47-73`,
`runtime/runtime.ts:2583-2637`.

```typescript
const connection = ManagedResource.make({
  tag: Connection,
  requirements: S.Struct({ url: S.String }),
  acquire: ({ url }) => connect(url),
  release: connection => connection.close(),
})
```

Represent resource availability as an Effect service whose lifetime follows
Model requirements. This is more composable than placing connection ownership in
individual React components.

### 17.5 Explicit Time Domains

**Source:** `runtime/runtime.ts:1833-1877`, `1959-1987`, `2314-2421`.

```text
update       = synchronous and ordered
Command      = microtask-started Effect fiber
render       = requestAnimationFrame-coalesced
```

Make these boundaries explicit. It becomes easier to reason about causality,
avoid effect reentrancy, and test each phase independently.

### 17.6 Stable Message Batch Semantics

**Source:** `runtime/runtime.ts:1999-2053`; tests in
`runtime/messageProcessing.test.ts`.

```typescript
// Messages dispatched while processing A are queued after the current batch.
process([A, B])
  // update(A) dispatches C
  // next batch is [B, C]
```

Batch snapshots prevent reentrant dispatch from starving already queued input.
This is useful for any central Effect event loop.

### 17.7 Typed Host Boundaries

**Source:** `packages/foldkit/src/port/port.ts:23-155`,
`runtime/runtime.ts:980-1148`, `3414-3493`.

```typescript
const HostSearch = Port.inbound(S.Struct({ query: S.String }))
const SearchResult = Port.outbound(S.Array(ResultSchema))
```

Decode inbound data and encode outbound data at the boundary. Do not expose the
internal Model or arbitrary dispatch function to a React host.

### 17.8 Separate State and Scene Testing

**Source:** `packages/foldkit/src/test/story.ts`, `test/internal.ts:263-338`,
`test/scene.ts:148-171`.

```typescript
Story.with(model)
  .send(Requested())
  .resolve(Load, Loaded(data))

Scene.start({ model, update, view })
  .getByRole('button', { name: 'Save' })
  .click()
```

One test layer should verify update/Effect semantics without running the browser;
another should verify the actual rendered interaction path.

### 17.9 Replay Without Executing Effects

**Source:** `packages/foldkit/src/devTools/store.ts:216-250`,
`runtime/runtime.ts:2200-2229`.

```text
historical Model = replay update from keyframe
historical render = no-op Message dispatch
Commands = never executed during replay
```

Keep debugging replay separate from live Effect execution. This prevents time
travel from sending network requests or mutating durable state.

### 17.10 Stable Identity for Nested Features

**Source:** `packages/foldkit/src/html/submodel.ts:103-138`,
`253-364`.

```typescript
h.submodel('settings-panel', settingsView, model.settings, liftMessage)
```

Require stable slot identity for nested state, subscriptions, cached views, and
DevTools paths. Treat identity as part of the feature composition contract.

---

## 18. Tradeoffs and Open Questions

### Strengths

- Effect scopes provide a clear ownership tree for asynchronous work.
- Messages make state transitions explicit, ordered, and replayable.
- Commands keep side effects visible without embedding them in views.
- Subscriptions and ManagedResources make lifecycle declarative.
- Ports provide a narrow, typed embedding boundary.
- Story and Scene testing cover both state and rendered behavior.
- DevTools replay and MCP integration benefit from Schema-defined Messages.

### Costs

- The architecture is not incremental: application code is expected to use the
  Foldkit runtime rather than mixing arbitrary local state into it.
- TypeScript cannot enforce purity inside `update`.
- The synchronous Message drain has no priority model.
- Concurrent Commands require explicit modeling when ordering matters.
- Submodel identity and message lifting add ceremony.
- Snabbdom means React's concurrent rendering model is not available.
- Mount failures have different error semantics from Commands and Subscriptions.
- Runtime IDs depend on DOM container IDs, which complicates multiple instances.
- Query strings do not directly provide repeated-key collection semantics.

### Questions for a React-Based Design

1. Should the central store publish through `useSyncExternalStore` rather than
   forcing React updates through an imperative callback?
2. Should Message reduction remain synchronous, or should Effect own a mailbox
   and expose urgent/background policies?
3. Can Model writes be immediate while derived recomputation and React
   notification are scheduled separately?
4. How should `startTransition` map to Effect-triggered invalidations?
5. Should Commands be serial by default, concurrent by default, or explicitly
   grouped by policy?
6. Can Foldkit-style ManagedResources be exposed through a React context without
   making React responsible for their cleanup?
7. Should host integration use typed Ports, a store API, or both?
8. Which replay guarantees are required when update reads nondeterministic or
   external values?

---

## 19. Key Files Reference

### Core Runtime

| File | Purpose |
|---|---|
| `packages/foldkit/src/runtime/runtime.ts` | Startup, Message queue, Commands, subscriptions, resources, render scheduling, crash handling, embed, disposal. |
| `packages/foldkit/src/runtime/browserListeners.ts` | Browser navigation and lifecycle listeners. |
| `packages/foldkit/src/runtime/preserveScheduler.ts` | Debounced HMR Model preservation. |
| `packages/foldkit/src/runtime/crashUI.ts` | Crash fallback view and reporting. |
| `packages/foldkit/src/runtime/messageProcessing.test.ts` | FIFO and reentrancy contracts. |

### Effects and Lifecycle

| File | Purpose |
|---|---|
| `packages/foldkit/src/command/index.ts` | Typed Commands and Message mapping. |
| `packages/foldkit/src/command/interruptible/interruptible.ts` | Keyed Command interruption. |
| `packages/foldkit/src/subscription/subscription.ts` | Model-dependent streams. |
| `packages/foldkit/src/subscription/fromEvent.ts` | Scoped browser event streams. |
| `packages/foldkit/src/managedResource/managedResource.ts` | Model-driven Effect services. |
| `packages/foldkit/src/mount/index.ts` | Element-bound Effect lifecycle. |

### Rendering and Composition

| File | Purpose |
|---|---|
| `packages/foldkit/src/html/index.ts` | HTML builder and DOM attributes. |
| `packages/foldkit/src/html/boundary.ts` | Runtime and Submodel dispatch boundaries. |
| `packages/foldkit/src/html/submodel.ts` | Nested view/message composition. |
| `packages/foldkit/src/snabbdom/init.ts` | VDOM patch lifecycle and hook ordering. |
| `packages/foldkit/src/render/render.ts` | `afterCommit` and `afterPaint`. |
| `packages/foldkit/src/route/parser.ts` | Bidirectional typed route parser/printer. |
| `packages/foldkit/src/route/transition.ts` | Route entry/exit/stayed policies. |
| `packages/foldkit/src/port/port.ts` | Typed host boundary. |

### Debugging and Testing

| File | Purpose |
|---|---|
| `packages/foldkit/src/devTools/store.ts` | History, keyframes, replay state. |
| `packages/foldkit/src/devTools/webSocketBridge.ts` | Runtime DevTools protocol. |
| `packages/devtools/src/overlay.ts` | Foldkit DevTools UI. |
| `packages/devtools-mcp/src/tools.ts` | MCP inspection and dispatch tools. |
| `packages/foldkit/src/test/story.ts` | Update/Command tests. |
| `packages/foldkit/src/test/scene.ts` | VDOM interaction tests. |

### Representative Examples

| Example | Demonstrates |
|---|---|
| `examples/counter` | Minimal Model/update/view runtime. |
| `examples/api-cache` | Async cache and Story/Scene tests. |
| `examples/auth` | Nested features, routing, and authentication flow. |
| `examples/embedding` | Typed Ports and host lifecycle. |
| `examples/charting` | Mount-based third-party DOM integration. |
| `examples/managed-resource-layer` | Effect Layer-backed resources. |
| `examples/route-transitions` | Entry, exit, and stayed navigation policies. |
| `examples/kanban` | Complex Submodel and Command composition. |
