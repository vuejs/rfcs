- Start Date: 2026-03-05
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: (leave this empty)

# Summary

Treat async reads and optimistic writes as states of Vue's reactive graph.

- A readonly `computed()` getter may return a Promise. Vue exposes its fulfilled
  result as the computed value and propagates pending and rejected states through
  derived computations.
- The existing `<Suspense>` component turns a pending render into fallback UI
  and keeps previously committed content visible during revalidation.
- A small `optimistic()` helper layers a tentative update over a ref or computed
  value, then reconciles it on success or rolls it back on failure.

The proposal adds no directive, component prop, SFC syntax, or separate memo
primitive. In particular, it does not add `<script async setup>`, `v-defer`, or
`<Suspense :with>`.

# Basic example

```vue
<script setup lang="ts">
import { computed, ref } from "vue"

type User = {
  id: number
  name: string
}

const userId = ref(1)

const user = computed(async () => {
  // Reactive dependencies are captured before the first await.
  const id = userId.value
  const response = await fetch(`/api/users/${id}`)

  if (!response.ok) {
    throw new Error(`Could not load user ${id}`)
  }

  return (await response.json()) as User
})

// Pending state composes through ordinary derived state. Neither `user` nor
// `heading` exposes a Promise or an intermediate `undefined` value.
const heading = computed(() => `Profile: ${user.value.name}`)
</script>

<template>
  <button type="button" @click="userId++">Next user</button>

  <Suspense>
    <UserProfile :heading="heading" :user="user" />

    <template #fallback>
      <UserProfileSkeleton />
    </template>
  </Suspense>
</template>
```

The first read has no settled value, so `<Suspense>` renders its fallback. When
`userId` changes later, Vue keeps the committed profile visible while the next
reactive answer is prepared, then commits the new profile as one update.

# Motivation

Vue currently has two separate models:

- synchronous values participate in the reactive graph; and
- asynchronous values are managed imperatively with loading refs, error refs,
  request identity checks, and watchers, or they block an entire component via
  async `setup()`.

This split makes both reads and writes harder to compose. Applications rebuild
the same state machines for initial loading, revalidation, stale responses,
optimistic updates, rollback, and concurrent mutations.

## Why `<Suspense>` remains incomplete

The current
[`<Suspense>` documentation](https://vuejs.org/guide/built-ins/suspense.html)
describes a boundary around two kinds of dependencies: components with async
`setup()` and async components. That is useful, but it is not yet a general
model for asynchronous reactive state.

### Dependencies are component-shaped

A Promise returned from `computed()` is currently just another value. Derived
computed values receive the Promise itself, and `<Suspense>` cannot observe it.
Async work must therefore move into component setup merely to participate in a
loading boundary.

This makes component boundaries carry data-flow semantics. Moving a request
between a parent, child, or composable can silently change which `<Suspense>`
owns it.

### Async setup is implicit and too coarse

Adding a top-level `await` to `<script setup>` implicitly turns the whole
component into an async dependency. The boundary waits for the complete setup
function even when only one derived region needs the result. The dependency is
not visible from the consuming template, and removing or relocating the `await`
changes boundary behavior again.

Async setup is still the right primitive for component initialization. It is not
a replacement for a reactive value that can refetch after mount.

### Revalidation is tied to tree shape

After a boundary has resolved, current re-entry behavior is coupled to replacing
the root node of its default slot. A data dependency changing inside an existing
component is not itself a Suspense event. Applications must manually decide
whether to clear content, retain stale content, or show a second loading state.

Tree shape should not decide whether a data update is coherent. The graph knows
which render read is waiting and whether a previously committed answer exists.

### Pending causes are difficult to inspect

The boundary exposes `pending`, `resolve`, and `fallback` events, but it does not
identify the source that is pending. A fallback delayed by one deeply nested
async component is difficult to trace, and refactoring can move that dependency
without changing the boundary template.

This RFC does not make every dependency explicit in markup. Instead, it requires
the runtime and DevTools to preserve source identity and show the computed or
component owner responsible for each pending generation.

### Mutations use a separate consistency model

Suspense coordinates missing read values, but it says nothing about a write that
crosses a network boundary. Optimistic UI currently requires an application to
snapshot state, apply a temporary value, reconcile the server result, restore the
snapshot on failure, and guard every step against overlapping mutations.

An optimistic value is not a loading fallback. It is a complete but tentative
answer. It should flow through the same derived graph immediately while retaining
enough identity to roll back or rebase safely.

### Loading and errors are only partially composed

`<Suspense>` intentionally does not catch errors. Async setup failures use
`onErrorCaptured` or `app.config.errorHandler`, while loading uses a separate
boundary. That separation is sound, but Promise-valued reactive state currently
participates in neither path without manual glue.

## Design direction

The graph, rather than `<Suspense>`, should own async state. A boundary only
projects that state into UI:

- pending without a settled answer renders fallback content;
- pending with a settled answer holds the last committed view;
- fulfillment commits a complete replacement;
- rejection follows Vue's existing error pipeline; and
- an optimistic layer is visible immediately, then commits, rebases, or rolls
  back as one identified mutation.

This direction is inspired by
[Solid 2.0's async reactivity](https://www.solidjs.com/blog/solid-2-0-rc-the-big-reveal),
but uses Vue's vocabulary and conventions: `computed()`, refs, `<Suspense>`, and
lowercase functional APIs.

An earlier version of this RFC proposed `<script async setup>`, `v-defer`, and
`<Suspense :with>`. Those APIs exposed the same underlying state at three syntax
layers and stopped composing when a value moved. This revision keeps the useful
motivation while moving the shared mechanism into reactivity.

The goals are:

1. Compose async and synchronous computed values without Promise plumbing.
2. Make `<Suspense>` react to data dependencies rather than component shape.
3. Never expose a partially available graph to a component render.
4. Keep the last committed view visible while a replacement is in flight.
5. Make optimistic updates deterministic under success, failure, and overlap.
6. Avoid defining a request cache, transport, or application data model.

# Detailed design

## Promise-aware `computed()`

The readonly getter form of `computed()` becomes Promise-aware. Its type is
conceptually changed to:

```ts
declare function computed<T>(
  getter: ComputedGetter<T>,
  debuggerOptions?: DebuggerOptions
): ComputedRef<Awaited<T>>
```

For example:

```ts
const count = computed(() => 1)
//    ^? ComputedRef<number>

const user = computed(async () => fetchUser())
//    ^? ComputedRef<User>

const maybeUser = computed(() => cachedUser.value ?? fetchUser())
//    ^? ComputedRef<User>
```

The writable form of `computed()` is unchanged. Promise-aware computed values
are readonly because there is no clear meaning for an asynchronous getter paired
with a synchronous setter.

Promise assimilation follows normal JavaScript thenable semantics. Nested
Promises are flattened in the same way as `Awaited<T>`.

Code that intentionally needs a Promise as the value can preserve it by placing
it inside another value:

```ts
const request = computed(() => ({ promise: fetchUser(userId.value) }))
//    ^? ComputedRef<{ promise: Promise<User> }>
```

## Computation lifecycle

An async computed value keeps three pieces of internal state:

- the last successfully settled value, if one exists;
- the current in-flight Promise, if one exists; and
- the latest rejection, if the current generation failed.

Evaluation remains lazy. Reading a dirty computed invokes its getter exactly as
today. If the result is a thenable, Vue records that Promise as the current
generation.

The observable states are:

| State           | Settled value | Result of a reactive read            |
| --------------- | ------------- | ------------------------------------ |
| Initial pending | No            | Propagate pending                    |
| Fulfilled       | Yes           | Return the value                     |
| Refreshing      | Yes           | Propagate pending for the new update |
| Optimistic      | Yes           | Return the layered value             |
| Rejected        | Maybe         | Propagate the error                  |

Pending is an internal graph condition, not a public value. A consumer never
receives the Promise, `undefined`, or the previous result as if it were current.

Only the latest generation may commit. If dependencies change while a request
is pending, Vue can start a new generation on the next read. Fulfillment or
rejection from an older generation is ignored. Disposing the computed value's
effect scope likewise prevents a late Promise from committing.

This proposal does not cancel obsolete work. A getter may implement cancellation
itself, but ignoring stale completion is required for correctness.

## Dependency tracking

Vue tracks reactive reads made during the synchronous invocation of the getter.
For an `async` function, this means reads before its first `await`:

```ts
const user = computed(async () => {
  const id = userId.value // tracked
  const response = await fetch(`/api/users/${id}`)
  return response.json()
})
```

Reactive reads made after an `await` are not tracked. This matches the point at
which the getter has returned control to Vue and avoids requiring async context
propagation. Inputs should be captured before the first `await`.

## Graph propagation

Pending and rejected states propagate through computed dependencies just as
invalidations do today:

```ts
const organization = computed(async () => fetchOrganization(orgId.value))
const owner = computed(() => organization.value.owner)
const label = computed(() => `${owner.value.name}'s organization`)
```

If `organization` is pending, `owner` and `label` do not cache an incomplete
value. The terminal reactive consumer is paused and retried after the Promise
settles. A rejection follows the same path as a synchronous error thrown by a
computed getter.

This is the key distinction from unwrapping a Promise only in the template:
derived state remains ordinary Vue code and carries no loading union.

## `<Suspense>` integration

When a component render reads a pending async computed value, the nearest
existing `<Suspense>` boundary registers that computation as a dependency.

- Before the first value settles, the boundary renders its `#fallback` slot.
- After content has committed, a new pending generation enters the boundary's
  pending state while preserving the currently committed tree.
- A pending read can re-enter the boundary regardless of whether the root VNode
  of the default slot was replaced. Revalidation is driven by data flow, not
  tree shape.
- The pending render is committed only after every async computed value read by
  that branch has settled.
- Nested boundaries capture reads from their own branches, as they do for async
  setup dependencies today.

The existing `timeout` prop and `pending`, `resolve`, and `fallback` events keep
their current roles. No Promise list, scoped-slot value, or new error slot is
added to `<Suspense>`.

Each registered dependency retains the computed instance, generation, component
owner, and nearest boundary as debugging metadata. DevTools can therefore show
what a boundary is waiting for without adding source names to the template API.

A component that reads an initially pending value without a `<Suspense>` ancestor
receives the same development warning and no-content behavior as a component
with async `setup()` and no boundary.

Multiple values require no coordination API:

```vue
<script setup lang="ts">
import { computed } from "vue"

const user = computed(async () => fetchUser())
const posts = computed(async () => fetchPosts())
</script>

<template>
  <Suspense>
    <Dashboard :posts="posts" :user="user" />

    <template #fallback>
      <DashboardSkeleton />
    </template>
  </Suspense>
</template>
```

The boundary discovers both reads through the graph and waits for both. Nested
`<Suspense>` boundaries can be used when the two regions should reveal
independently.

## Promise props

A component can adapt a Promise prop with the same primitive:

```vue
<script setup lang="ts">
import { computed } from "vue"

const props = defineProps<{ source: Promise<User> }>()
const user = computed(() => props.source)
</script>

<template>
  <UserProfile :user="user" />
</template>
```

No directive or scoped slot is required. The boundary is supplied by whichever
ancestor owns the loading experience.

## Optimistic updates

Async reads need a settled-answer model. Mutations need the complementary
ability to publish a tentative answer immediately and either reconcile or remove
it later.

This RFC adds one lowercase functional API:

```ts
interface OptimisticOptions<T, Result> {
  update: (current: T) => T
  run: () => Result | PromiseLike<Result>
  commit?: (current: T, result: Awaited<Result>) => T
}

declare function optimistic<T, Result>(
  target: Ref<T>,
  options: OptimisticOptions<T, Result>
): Promise<Awaited<Result>>
```

`target` may be a mutable ref or a readonly computed ref. `optimistic()` does not
assign to a readonly computed value; it attaches an identified overlay to that
reactive source. Derived computed values and component renders observe the
overlay through ordinary reads.

```vue
<script setup lang="ts">
import { computed, optimistic } from "vue"

type Todo = {
  id: string
  title: string
  pending?: boolean
}

const todos = computed(async () => fetchTodos())

function addTodo(title: string) {
  const temporaryId = crypto.randomUUID()

  return optimistic(todos, {
    update: (current) => [
      ...current,
      { id: temporaryId, title, pending: true }
    ],
    run: () => api.createTodo({ title }),
    commit: (current, saved) =>
      current.map((todo) => (todo.id === temporaryId ? saved : todo))
  })
}
</script>

<template>
  <Suspense>
    <ul>
      <li v-for="todo in todos" :key="todo.id">
        {{ todo.title }}
        <small v-if="todo.pending">Saving...</small>
      </li>
    </ul>

    <template #fallback>
      <TodoListSkeleton />
    </template>
  </Suspense>
</template>
```

The operation follows this lifecycle:

1. The target must have a settled value. Calling `optimistic()` before the first
   answer settles returns a rejected Promise and does not start `run`.
2. `update` is applied as a tentative layer and becomes visible immediately.
   It does not enter `<Suspense>` pending state because it is a complete
   provisional answer.
3. `run` performs the mutation and may cross an async boundary.
4. On fulfillment, `commit` receives this layer's optimistic value and the
   fulfilled result. Its return value replaces the tentative layer. If `commit`
   is omitted, the value produced by `update` becomes committed.
5. On rejection, the layer is removed, the previous answer becomes visible
   again, and the Promise returned by `optimistic()` rejects with the same
   reason.

Both `update` and `commit` must be pure and replayable. This permits deterministic
overlap. Vue stores optimistic layers in invocation order. If an earlier
operation settles while later layers exist, Vue updates the base and replays the
later layers. If a later operation settles first, its layer remains ordered but
is marked fulfilled until earlier layers can be folded. Rejecting any layer
removes only that layer and rebases the remaining ones.

When the underlying ref changes or an async computed generation fulfills, that
answer becomes the new base and active optimistic layers are replayed over it.
When no layers remain, the target behaves exactly as it did before the mutation.

An active optimistic layer also supplies a complete answer while its underlying
computed source is revalidating. The source remains pending for diagnostics, but
that pending state does not hide or delay the tentative UI.

This keeps mutation policy deliberately small. The helper does not define an API
transport, cache key, form action, retry policy, or persistence format.

## Error handling

`<Suspense>` remains a loading boundary, not an error boundary. If an async
computed Promise rejects, Vue reports the rejection through the existing
`onErrorCaptured` and `app.config.errorHandler` pipeline.

An initial rejection prevents the pending branch from committing. A rejection
during refresh leaves the previous committed tree in place and reports the
error. The rejected computation can run again after one of its tracked inputs is
invalidated.

This RFC does not add `#error`, `@error`, retry, or source-level error state to
`<Suspense>`.

A mutation rejection is separate from a read rejection. `optimistic()` first
rolls back its layer, then rejects its returned Promise so the event handler or
calling library can display an error. `<Suspense>` does not replace the retained
UI with loading or error content for that mutation.

## Server rendering and hydration

During SSR, a pending read causes the server renderer to await the current
generation and retry the affected branch, using the same boundary ownership as
async setup dependencies.

This proposal does not define request caching or data serialization. A client
may therefore start the computation again during hydration. If server-rendered
content already exists, it is treated as the committed view and remains visible
until the client generation settles. A userland cache can avoid duplicate work.

## Existing async setup

Top-level `await` in `<script setup>` and async `setup()` retain their current
syntax and behavior. They continue to register the whole component as an async
dependency. This RFC adds a finer-grained reactive path; it does not replace or
rename async setup.

## Non-goals

This RFC deliberately does not define:

- request caching, deduplication, preloading, or serialization;
- a router, server-action, transport, or form-submission protocol;
- draft mutation semantics for arbitrary reactive objects and collections;
- automatic network cancellation;
- async iterator or streaming-value semantics;
- a source-level `isPending()` API; or
- general stabilization of every experimental `<Suspense>` behavior.

Boundary-level loading indicators can use the existing `<Suspense>` events. A
future RFC can propose source-level status if concrete use cases require it.

# Drawbacks

- **This changes existing behavior.** A readonly computed getter that
  intentionally returns a Promise currently exposes that Promise. Under this
  proposal it exposes the fulfilled value. The wrapper escape hatch is simple,
  but finding intentional Promise-valued computed state requires an ecosystem
  audit.
- **The implementation reaches into the reactive scheduler and renderer.** A
  pending graph must be held without committing partial work, retried after
  settlement, and connected to the correct boundary.
- **Optimistic layering is core state machinery.** Layers must be replayable,
  ordered across concurrent operations, and rebased when their underlying source
  changes. This is more complex than snapshot-and-restore for one mutation.
- **`optimistic()` is new public API.** It avoids a larger action or cache
  protocol, but still adds a concept developers must learn for writes.
- **Async causes remain implicit in the template.** A `<Suspense>` boundary does
  not list every computation it may wait for. DevTools should show pending
  computations and their owner stacks to make this inspectable.
- **Only pre-`await` dependencies are tracked.** This rule is predictable but
  requires getters to capture reactive inputs early.
- **Cancellation is not automatic.** Ignoring an obsolete answer prevents stale
  commits but does not save network or server work.

# Alternatives

## Keep the previous three primitives

`<script async setup>`, `v-defer`, and `<Suspense :with>` make async work visible
at particular syntax sites. They also create three overlapping concepts, add
compiler and runtime surface, and stop composing when the value moves to a
different layer. The graph still needs async semantics for derived state.

## Add `computedAsync()`

A separate function is easier to introduce without changing Promise-valued
`computed()` behavior. It also makes async state look like a parallel reactivity
system and forces callers to choose a different primitive when an implementation
becomes async. This remains a compatibility fallback if changing `computed()`
cannot be shipped safely, but it is not the preferred API.

## Add an `<Await>` component or `v-defer`

Template-only unwrapping is explicit and can be implemented without changing
computed values. It does not compose through derived state and requires Promise
plumbing across component boundaries.

## Port Solid's action APIs directly

An `action()` transaction plus separate optimistic signals or stores can model
more complex mutation workflows. In JavaScript, preserving implicit transaction
context across `await` requires a transform, async-context facility, or generator
protocol. It also adds several public primitives before Vue has established the
smaller common case.

The proposed `optimistic()` options make each tentative write and async operation
explicit in one call. A general action protocol can be added later without
changing the overlay semantics.

## Add `useOptimistic()` around each source

A composable could return a second ref and a mutation function. This is viable in
userland, but creates a parallel value that must be threaded through every
consumer. Attaching the overlay to the original reactive source keeps existing
derived computed values and components unchanged.

## Keep async state in userland

Libraries can expose `{ data, pending, error }` refs and implement cancellation
or caching policies. They cannot atomically hold Vue's render graph or register
arbitrary derived reads with `<Suspense>` without a core protocol.

## Keep the status quo

Async setup remains suitable for one-time initialization, and applications keep
managing reactive refetches separately. This has no compatibility cost but leaves
the composition problem unsolved.

# Adoption strategy

Existing async setup components and `<Suspense>` templates require no migration.
Applications can replace manual loading state incrementally, one computed chain
at a time.

`optimistic()` is opt-in and can replace mutation-specific snapshot and rollback
code one operation at a time. Existing refs, computed values, and event handlers
remain the public data path; no component migration is required.

Before enabling Promise assimilation by default, the ecosystem must be scanned
for readonly computed values whose public value is intentionally a Promise. A
development warning can identify these sites in an earlier release. Intentional
Promise values migrate to a wrapper object or `shallowRef<Promise<T>>()`.

If the audit shows that a minor release would be unsafe, the behavior should be
enabled only in the next major or behind an explicit compatibility flag. There
is no reliable codemod because intent cannot be inferred from a Promise-returning
getter alone.

DevTools should identify async computed values, their current generation, and
the `<Suspense>` boundary waiting on them before the feature is considered
stable.

# Unresolved questions

1. What should a direct `.value` read do before the first result settles when it
   occurs outside a managed reactive consumer? Returning `undefined` would make
   the type dishonest, while exposing the Promise would break graph composition.
2. Should `watch()` and `watchEffect()` pause on pending reads, or should the
   first version support only computed chains terminating in component render
   effects?
3. Is the compatibility risk small enough for the experimental Vue 3 Suspense
   feature, or must Promise assimilation wait for the next major version?
4. Should the first `optimistic()` version accept only refs and computed refs, or
   also provide draft updaters for reactive objects, arrays, `Map`, and `Set`?
5. How should direct writes to a mutable ref interleave with optimistic layers?
   Treating each write as a new base and replaying active layers is the proposed
   rule, but it needs validation against concurrent form and store updates.
6. Should a future cleanup hook receive an `AbortSignal`, or should cancellation
   remain entirely userland policy?
