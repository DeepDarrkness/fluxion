# Foldkit and Effect Atom: Reusable Architecture Ideas

> A comparison of the architectural decisions behind React + Effect Atom and
> Foldkit. This document is not a recommendation to adopt either library
> directly. It extracts the ideas each author chose, the benefits those choices
> provide, and the tradeoffs they accept.

## Executive Summary

Effect Atom and Foldkit share Effect as their foundation, but use it at
different architectural boundaries.

Effect Atom makes Effect-backed reactive cells the primary state abstraction:

```text
Effect computation -> Atom registry -> host framework subscription -> React
```

Foldkit makes an Elm-style application loop the primary abstraction:

```text
Message -> update(Model, Message) -> Model + Commands -> runtime
```

The most reusable ideas are complementary:

- Effect Atom demonstrates how to make asynchronous computations, service
  Layers, caching, Suspense, and fine-grained invalidation ergonomic.
- Foldkit demonstrates how to make state transitions, effect ownership,
  lifecycle, replay, and testing explicit.
- The main design choice is where to place the authoritative boundary: inside a
  reactive graph of independently writable cells, or inside an explicit event
  and transition runtime.

Neither choice is universally superior. Each author has optimized for a
different failure mode.

## 1. The Chosen Invariants

### Effect Atom: local independence and reactive composition

Effect Atom treats an atom as a reactive container for one value. Atoms can be
read, written, derived, composed into families, or backed by an Effect or
Stream.

The key invariant is local dependency correctness: a component subscribes to
the atoms it reads, and derived atoms track the atoms they read during
evaluation.

This makes feature ownership local. Adding a new state cell does not require
editing a central reducer or registering a global Message variant.

**Benefits**

- Features can evolve independently.
- Fine-grained subscriptions avoid broad component updates.
- Derived state follows the same abstraction as source state.
- Existing React applications can adopt one atom at a time.

**Tradeoffs**

- There is no required application-wide catalog of state transitions.
- Writes can originate wherever a writable atom is available.
- Understanding the complete causal graph requires following atom reads,
  writes, and invalidation metadata across files.
- Application-level ordering is not the primary abstraction.

**References**

- Effect Atom API, constructors and combinators: [Atom.ts](https://tim-smart.github.io/effect-atom/atom/Atom.ts.html#constructors)
- Effect Atom comparison, “Many Atoms vs One Model”: [foldkit-vs-react-effect-atom#many-atoms-vs-one-model](https://foldkit.dev/react/foldkit-vs-react-effect-atom#many-atoms-vs-one-model)
- Effect Atom hooks: [Hooks.ts](https://tim-smart.github.io/effect-atom/atom-react/Hooks.ts.html#hooks)

### Foldkit: one explicit transition path

Foldkit requires one Model, one Message union, and one update function per
application or Submodel. Every canonical state change enters through that
function.

The key invariant is causal visibility:

```text
Message -> update -> next Model + Commands
```

`M.tagsExhaustive` makes the Message union a compile-time checklist.

**Benefits**

- State transitions have named, inspectable causes.
- Missing transition handling becomes a type error.
- State changes can be replayed when inputs are deterministic.
- Commands can be inspected without executing their Effects.
- DevTools and tests can operate on Messages and Models rather than DOM
  implementation details.

**Tradeoffs**

- Simple local mutations require more types and ceremony.
- A central update function grows unless the application introduces Submodels.
- Canonical state cannot be freely mutated by any component.
- The architecture is not incrementally adoptable inside a normal React tree.

**References**

- Foldkit architecture, “The Loop”: [core/architecture#the-loop](https://foldkit.dev/core/architecture#the-loop)
- Local implementation analysis: `knowledge/foldkit_arch.md:36-97`
- Update contract and purity boundary: `knowledge/foldkit_arch.md:138-157`
- Comparison, “Foldkit: one Message union, one update”: [foldkit-vs-react-effect-atom#foldkit-state](https://foldkit.dev/react/foldkit-vs-react-effect-atom#foldkit-state)

## 2. Ideas from Effect Atom

### 2.1 Treat Effect computations as reactive values

Effect Atom makes an Effect directly usable as the producer of an atom:

```ts
const userAtom = runtime.atom(
  Effect.gen(function* () {
    const api = yield* Api
    return yield* api.getUser()
  }),
)
```

The result is represented as a typed `Result`, rather than requiring each
feature to invent its own loading and failure protocol.

This is a strong idea for any React-oriented Effect architecture: asynchronous
work should be a first-class value with an explicit lifecycle, not an
imperative callback hidden in a component.

**What it buys**

- Typed success and failure channels.
- A standard initial/loading/success/failure vocabulary.
- Lazy evaluation and runtime caching.
- A direct bridge from Effect services to view-readable state.

**What it costs**

- Async state lives in the atom registry, potentially outside a canonical
  application Model.
- The registry becomes responsible for cache identity, refresh, and disposal.
- Mutations and query invalidation need explicit conventions such as
  `reactivityKeys`.

**References**

- `Atom.make` signatures: [Atom.ts#make](https://tim-smart.github.io/effect-atom/atom/Atom.ts.html#make)
- Result union and waiting state: [Result.ts#result-type-alias](https://tim-smart.github.io/effect-atom/atom/Result.ts.html#result-type-alias)
- Effect Atom README, “Working with Effects”: [github.com/tim-smart/effect-atom#working-with-effects](https://github.com/tim-smart/effect-atom#working-with-effects)
- Comparison, async state: [foldkit-vs-react-effect-atom#async-state](https://foldkit.dev/react/foldkit-vs-react-effect-atom#async-state)

### 2.2 Use Effect Layers as application dependency injection

`Atom.runtime(layer)` allows atom computations and function atoms to access
Effect services through the normal Effect environment.

This keeps API clients, configuration, storage, RPC, and platform services
typed and replaceable. It also avoids making React Context the primary
dependency-injection mechanism.

**Reusable lesson**

Use Effect Layers at the runtime boundary, then let individual computations
declare only the services they require. This preserves testability and avoids
passing service objects through component props.

**Tradeoff**

Layer construction and scope ownership must be understood. A runtime-wide
Layer, an atom-scoped service, and a component-scoped resource have different
lifetime semantics.

**References**

- `Atom.runtime`: [Atom.ts#runtime](https://tim-smart.github.io/effect-atom/atom/Atom.ts.html#runtime)
- README, “Working with Effect Services / Layers”: [effect-atom README](https://github.com/tim-smart/effect-atom#working-with-effect-services--layers)
- Registry Layer API: [Registry.ts#layers](https://tim-smart.github.io/effect-atom/atom/Registry.ts.html#layers)

### 2.3 Make cache identity explicit and runtime-scoped

Effect Atom supports `Atom.family` for stable keyed atoms and provides runtime
helpers for async atoms, HTTP APIs, RPC, refresh, and stale state.

The important idea is not merely “add a cache.” It is that cache identity and
resource ownership must be scoped. A query should not accidentally share work
between unrelated application instances or server requests.

**What it buys**

- Request deduplication.
- Reuse across components reading the same keyed value.
- Lazy cleanup when values are no longer observed.
- A natural place for stale-while-revalidate policies.

**Tradeoffs**

- Cache behavior can become implicit from the perspective of a state-machine
  test.
- Key equality, refresh invalidation, and eviction are part of correctness.
- A cached query result is not automatically a named application event.

**References**

- `Atom.family`: [Atom.ts#family](https://tim-smart.github.io/effect-atom/atom/Atom.ts.html#family)
- Atom runtime model: [Atom.ts#atomruntime-interface](https://tim-smart.github.io/effect-atom/atom/Atom.ts.html#atomruntime-interface)
- Comparison, async runtime capabilities: [foldkit-vs-react-effect-atom#atom-async](https://foldkit.dev/react/foldkit-vs-react-effect-atom#atom-async)

### 2.4 Keep reactivity fine-grained

The registry tracks atom dependencies and notifies consumers of changed atoms.
This allows a large application state domain to be projected into small
renderer-facing values.

This is valuable even in a centralized architecture. A single canonical Model
does not require every React component to subscribe to the whole Model. A
projection layer can expose atom-like or selector-like views of canonical
state.

**Reusable lesson**

Centralized transition ownership and fine-grained rendering subscriptions are
not opposites. They can be separate layers:

```text
canonical state -> derived projections -> component subscriptions -> React
```

The tradeoff is maintaining a coherent snapshot boundary between the
transition runtime and the reactive graph.

**References**

- Atom context dependency API: [Atom.ts#context-interface](https://tim-smart.github.io/effect-atom/atom/Atom.ts.html#context-interface)
- Registry subscription API: [Registry.ts#registry-interface](https://tim-smart.github.io/effect-atom/atom/Registry.ts.html#registry-interface)
- Comparison, “One Timeline vs Many Cells”: [foldkit-vs-react-effect-atom#one-timeline-vs-many-cells](https://foldkit.dev/react/foldkit-vs-react-effect-atom#one-timeline-vs-many-cells)

## 3. Ideas from Foldkit

### 3.1 Separate state transitions from Effect execution

Foldkit’s `update` function returns Commands but does not execute them. A
Command contains a name, arguments, and an Effect that eventually produces a
Message.

This separation is one of Foldkit’s strongest ideas:

```text
update = synchronous decision
Command = inspectable Effect description
runtime = Effect execution and Message delivery
```

**What it buys**

- Effects cannot accidentally run during a state transition.
- Commands can be asserted without network mocks.
- Async results re-enter through the same Message channel.
- Runtime policies such as concurrency, interruption, and retries have a
  named home.

**What it costs**

- An otherwise simple callback becomes a Message, a Command, and one or more
  result Messages.
- Ordering between concurrent Commands must be modeled explicitly.
- TypeScript does not technically prevent side effects inside `update`; the
  purity boundary is enforced by API shape and convention.

**References**

- Command documentation: [core/commands#overview](https://foldkit.dev/core/commands#overview)
- Command value and `Command.define`: `packages/foldkit/src/command/index.ts:1-190`
- Command execution and concurrency analysis: `knowledge/foldkit_arch.md:310-355`

### 3.2 Make lifecycle model-driven

Foldkit separates several lifecycle primitives:

- Commands for one-shot work.
- Subscriptions for Model-dependent streams.
- Managed Resources for long-lived acquired services.
- Mounts for work tied to a concrete DOM element.

This classification is valuable because it prevents every external activity
from becoming an ad hoc component effect.

**What it buys**

- Resource ownership is visible from the declaration.
- Cleanup follows Effect scopes and interruption.
- A WebSocket or event listener can follow Model state rather than a stale
  component closure.
- One-shot work and long-running work have different APIs.

**What it costs**

- More concepts to learn.
- Model dependency extraction and equivalence become part of lifecycle
  correctness.
- Lifecycle transitions may generate additional Messages and state branches.

**References**

- Architecture lifecycle definitions: [core/architecture#the-loop](https://foldkit.dev/core/architecture#the-loop)
- Subscriptions source: `packages/foldkit/src/subscription/subscription.ts:1-150`
- Subscription runtime analysis: `knowledge/foldkit_arch.md:359-395`
- Managed Resource analysis: `knowledge/foldkit_arch.md:399-439`
- Mount lifecycle analysis: `knowledge/foldkit_arch.md:515-551`

### 3.3 Use typed Messages as the application protocol

Foldkit treats Messages as facts about what happened, not imperative setters.
Commands are named imperatives; result Messages are named facts.

This distinction improves observability and makes asynchronous workflows
explicit:

```text
ClickedLoadUser -> FetchUser Command
FetchUser completes -> SucceededLoadUser Message
```

**What it buys**

- Intent and outcome are distinguishable.
- DevTools can display the causal chain.
- Tests can resolve a Command with an explicit result.
- Exhaustiveness exposes missing state-machine branches.

**What it costs**

- Message design becomes important API design.
- Low-value events such as every pointer movement can flood history unless
  excluded or modeled specially.
- Not every UI event should necessarily become a durable domain event.

Foldkit’s history is bounded debugging history, not event sourcing. This is an
important limitation and a useful boundary: replayable UI Messages should not
automatically be treated as long-lived business events.

**References**

- Foldkit Message loop: [core/architecture#the-loop](https://foldkit.dev/core/architecture#the-loop)
- DevTools history and replay: `knowledge/foldkit_arch.md:677-705`
- Message/event distinction and bounded replay: `knowledge/foldkit_vs_effract.md:432-465`

### 3.4 Build testing around values, not mocks

Foldkit’s Story tests execute `update` directly. Commands are recorded as
pending values and resolved with chosen result Messages. Scene tests exercise
the real view and Message path without requiring a browser DOM.

This provides two useful levels:

```text
Story = state transition and Command semantics
Scene = rendered interaction and accessibility path
```

**What it buys**

- Fast deterministic workflow tests.
- No need to intercept HTTP calls to prove that a Command was requested.
- Clear separation between state-machine failures and rendering failures.
- Tests describe the user-visible causal story.

**What it costs**

- The test framework must understand the runtime’s Command semantics.
- A pure state-machine test does not prove the real Effect implementation.
- Scene tests validate Foldkit’s renderer, not React behavior.

**References**

- Story and Scene analysis: `knowledge/foldkit_arch.md:708-751`
- Foldkit comparison testing section: [foldkit-vs-react-effect-atom#testing](https://foldkit.dev/react/foldkit-vs-react-effect-atom#testing)
- Command testing documentation: [core/commands#testable-by-design](https://foldkit.dev/core/commands#testable-by-design)

### 3.5 Make runtime timing explicit

Foldkit intentionally separates three timing domains:

```text
update       = synchronous and ordered
Command      = microtask-started Effect fiber
render       = requestAnimationFrame-coalesced
```

The runtime uses a FIFO Message queue and prevents recursive update execution.
It can yield after a time budget, but this is a fairness mechanism rather than
a React-style priority scheduler.

**What it buys**

- Easier reasoning about causality and reentrancy.
- Command results cannot re-enter the current update stack.
- Multiple state transitions can be processed before one coalesced render.

**What it costs**

- The queue has no built-in urgent/background priority model.
- A large synchronous update can still block the browser.
- The renderer and state runtime have different notions of “completed.”

**References**

- Message queue and reentrancy: `knowledge/foldkit_arch.md:222-271`
- Yield behavior: `knowledge/foldkit_arch.md:273-305`
- Timing domains: `knowledge/foldkit_arch.md:293-306`
- Comparison, rendering timeline: [foldkit-vs-react-effect-atom#one-timeline-vs-many-cells](https://foldkit.dev/react/foldkit-vs-react-effect-atom#one-timeline-vs-many-cells)

## 4. Combined Design Lessons

The strongest combined architecture would preserve the authors’ useful
separations instead of choosing one abstraction for every concern.

### Canonical state versus projected state

Use a centralized transition owner for state that needs explicit causality,
workflow semantics, or replay:

```text
Message -> update -> canonical Model
```

Use a fine-grained reactive graph for derived values and renderer-facing
projections:

```text
canonical Model -> projections -> React subscriptions
```

This gives Foldkit’s transition visibility without requiring every component
to rerender from a large Model.

### Feature state versus ephemeral UI state

Not every value needs a Message union. A practical split is:

```text
workflow/domain state -> explicit Messages and update
derived state          -> reactive projections
ephemeral local state  -> local atoms or component state
```

The cost is having more than one state mechanism, but that cost is preferable
to forcing transient hover or input state through a heavyweight application
protocol.

### Effect ownership outside speculative React render

Effect Atom makes Effects easy to read from React, but long-lived resource
acquisition should still occur in committed runtime scopes rather than as an
uncontrolled consequence of speculative rendering.

Foldkit provides the stronger default here: Commands, Subscriptions, and
Managed Resources execute outside view construction. A React bridge should
retain that boundary and use `useSyncExternalStore` only to read committed
snapshots.

### Typed invalidation versus named Messages

Effect Atom’s reactivity keys are useful for query invalidation. Foldkit’s
Messages are useful for causal state transitions.

They solve different problems:

```text
Message       = what happened in the application protocol
reactivity key = what cached computation should refresh
```

Combining them requires care. A mutation can produce a named result Message and
also invalidate a query projection, but the invalidation should not replace
the domain fact.

### Runtime scope per application instance

Both systems demonstrate that runtime identity matters. A root runtime should
own its registry, scopes, caches, subscriptions, and cleanup. Module-global
state is convenient but undermines server isolation, testing, and multiple
mounts.

**References**

- Effect Atom registry model: [Registry.ts#registry-interface](https://tim-smart.github.io/effect-atom/atom/Registry.ts.html#registry-interface)
- Foldkit root scope: `knowledge/foldkit_arch.md:161-218`
- Foldkit embedding and typed host boundary: `knowledge/foldkit_arch.md:638-673`

## 5. Author Tradeoff Summary

| Concern | Effect Atom’s choice | Foldkit’s choice |
|---|---|---|
| Primary abstraction | Reactive atom | Model and Message loop |
| State ownership | Distributed by feature/cell | Centralized by Model/update |
| Effect location | Atom computation, function atom, or subscription | Command, Subscription, Mount, or Resource |
| Async state | Registry-managed `Result` | Model-managed async value plus Messages |
| Rendering | Host framework, especially React | Foldkit virtual DOM and runtime |
| Adoption | Incremental | Whole-framework commitment |
| React ecosystem | Preserved | Replaced or used only at embedding boundaries |
| Invalidation | Dependency graph and keys | Model changes and subscription dependency extraction |
| Testing | Atom/Effect tests plus host-framework tests | Story and Scene runtime tests |
| Debugging | Current atom values and registry state | Named Message timeline and replay |
| Main optimization | Locality and fine-grained updates | Causal visibility and uniformity |
| Main cost | Distributed causality | Centralized ceremony and framework commitment |

## Conclusion

The useful question is not “which library should replace the other?” It is
which invariant a feature needs.

Choose Effect Atom’s ideas when the problem is reactive composition, async data,
keyed caching, service injection, and fine-grained React updates.

Choose Foldkit’s ideas when the problem is explicit workflows, lifecycle
ownership, deterministic testing, named effects, state transition auditing,
and replayable application behavior.

The most promising synthesis is:

```text
Effect runtime
  -> typed Messages and ordered feature transitions
  -> canonical state
  -> fine-grained reactive projections
  -> React via stable external-store snapshots

Effect scopes
  -> Commands
  -> Subscriptions
  -> Managed Resources
  -> cache and query lifecycles
```

That preserves Foldkit’s explicit causality, Effect Atom’s reactive ergonomics,
and React’s responsibility for rendering and scheduling.

## Source and Line-Reference Notes

The Foldkit implementation references in this report are based on the local
analysis document at the analyzed Foldkit commit `17d121ad`:

- `knowledge/foldkit_arch.md`
- `knowledge/foldkit_vs_effract.md`

The Effect Atom API site is generated documentation. Its pages expose stable
module, symbol, and section anchors but do not expose stable line-numbered
source listings. Those references therefore use exact module and heading
anchors. Foldkit GitHub source references use file and line ranges from the
numbered source listing.
