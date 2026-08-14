- Start Date: 2026-03-05
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: (leave this empty)

# Summary

Allow the getter passed to the existing `computed()` API to return a Promise.
Vue treats the fulfilled result as the computed value and carries pending and
rejected states through the reactive graph. The existing `<Suspense>` component
turns an unresolved read into fallback UI.

This proposal adds no public runtime function, directive, component prop, or SFC
syntax. In particular, it does not add `<script async setup>`, `v-defer`,
`<Suspense :with>`, or a separate memo API.

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

This split makes a common data flow unnecessarily difficult. A Promise returned
from `computed()` is currently just another value. Downstream computed values
receive the Promise itself, and `<Suspense>` cannot observe it. Applications
therefore rebuild an async state machine around every request.

Async `setup()` is useful for component initialization, but it is too coarse for
reactive refetching. It also does not solve composition: a child receiving a
Promise prop, or a computed value derived from asynchronous data, still needs a
second state model.

An earlier version of this RFC proposed three new primitives for these cases:
`<script async setup>`, `v-defer`, and `<Suspense :with>`. They exposed the same
underlying state at three different layers and required developers to choose a
different syntax whenever async work moved across a component boundary.

This revision instead makes async state part of the graph that already connects
Vue state to the renderer. It is inspired by
[Solid 2.0's async reactivity](https://www.solidjs.com/blog/solid-2-0-rc-the-big-reveal),
but uses Vue's existing vocabulary and boundaries: `computed()`, `<Suspense>`,
and Vue's error handling pipeline.

The goals are:

1. Compose async and synchronous computed values without Promise plumbing.
2. Use one existing loading boundary for setup and reactive async work.
3. Never expose a partially available graph to a component render.
4. Keep the last committed view visible while a replacement is in flight.
5. Avoid defining a request cache, data-fetching library, or mutation protocol.

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
- The pending render is committed only after every async computed value read by
  that branch has settled.
- Nested boundaries capture reads from their own branches, as they do for async
  setup dependencies today.

The existing `timeout` prop and `pending`, `resolve`, and `fallback` events keep
their current roles. No Promise list, scoped-slot value, or new error slot is
added to `<Suspense>`.

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
- mutation actions, optimistic state, or form submission state;
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
4. Should a future cleanup hook receive an `AbortSignal`, or should cancellation
   remain entirely userland policy?
