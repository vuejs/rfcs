- Start Date: 2026-03-05
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: (leave this empty)

# Summary

Stabilize `<Suspense>` by introducing explicit, composable primitives for async rendering:

1. **`<script async setup>`** - Explicit opt-in for async component setup (replacing implicit detection of top-level `await`)
2. **`v-defer` directive** - Template-level promise unwrapping with deferred rendering
3. **`<Suspense :with>` prop** - Explicit promise binding on Suspense boundaries with resolved value provision via scoped slots

These three primitives address the core issues preventing Suspense stabilization: implicit async dependency detection, lack of visibility into what causes suspension, and inconsistent patterns between setup-phase and render-phase async handling.

# Basic example

## Pattern A: Self-contained async (`<Suspense :with>`)

The component creates promises and manages its own Suspense boundary. No async setup required.

```vue
<script setup lang="ts">
const userData = fetchUser(userId)
const posts = fetchPosts(userId)
// Promises are created but NOT awaited
</script>

<template>
  <Suspense :with="{ userData, posts }">
    <template #default="{ userData, posts }">
      <UserProfile :user="userData" />
      <PostList :posts="posts" />
    </template>
    <template #fallback>
      <LoadingSkeleton />
    </template>
  </Suspense>
</template>
```

## Pattern B: Delegated async (`<script async setup>`)

The component declares itself as async. The parent provides the Suspense boundary.

```vue
<!-- AsyncChild.vue -->
<script async setup lang="ts">
const data = await fetchData()
</script>

<template>
  <div>{{ data }}</div>
</template>
```

```vue
<!-- Parent.vue -->
<template>
  <Suspense>
    <AsyncChild />
    <template #fallback>
      <p>Loading...</p>
    </template>
  </Suspense>
</template>
```

## Pattern C: Promise-as-prop (`v-defer`)

A reusable child component receives a Promise as a prop and defers its own rendering.

```vue
<!-- DataRenderer.vue -->
<script setup lang="ts">
const { source } = defineProps<{ source: Promise<string> }>()
</script>

<template v-defer="source as result">
  <p>{{ result }}</p>
</template>
```

```vue
<!-- Parent.vue -->
<script setup lang="ts">
const message = fetchMessage()
</script>

<template>
  <Suspense>
    <DataRenderer :source="message" />
    <template #fallback>
      <p>Loading...</p>
    </template>
  </Suspense>
</template>
```

# Motivation

## Current problems with `<Suspense>`

### 1. Implicit async dependency detection

In the current implementation, any `<script setup>` containing a top-level `await` implicitly becomes an async component and an async dependency of the nearest `<Suspense>`. This has several problems:

- **Surprising behavior**: Adding a single `await` to a component silently changes how it renders and requires a `<Suspense>` ancestor. Removing the `await` silently removes the dependency. This is easy to introduce or break accidentally.
- **Hard to debug**: When a `<Suspense>` shows its fallback longer than expected, there is no straightforward way to identify which descendant(s) are still pending.
- **Refactoring hazard**: Moving async logic between components can silently break Suspense boundaries.

### 2. No explicit connection between Suspense and its causes

The current `<Suspense>` has no API for declaring what it is waiting for. It implicitly collects all async descendants. This makes the template unreadable - you cannot see at a glance what a particular `<Suspense>` is waiting for.

### 3. Inconsistent async patterns

There are two fundamentally different phases where async can occur:

- **Setup phase**: The component's `setup()` is async (data fetching before first render)
- **Render phase**: The component's setup is synchronous, but rendering depends on a Promise that resolves later

Current Vue only supports the setup-phase pattern via `<Suspense>`. There is no built-in primitive for render-phase promise handling. Users resort to manual `ref` + `.then()` patterns or third-party composables, which don't integrate with Suspense boundaries.

### 4. Lack of composability

There is no standard way to pass a Promise through the component tree and have it integrate with Suspense. Patterns like "parent creates a promise, child renders the result" require manual wiring that is error-prone and doesn't participate in Suspense coordination.

## Design goals

1. **Explicitness**: Every async dependency should be visible in the code. A reader should be able to trace the cause of suspension from template to source.
2. **Separation of phases**: Setup-phase async and render-phase async should have distinct, purpose-built primitives.
3. **Composability**: Promises should be passable as props and composable across component boundaries while maintaining Suspense integration.
4. **Consistency**: All async patterns should integrate uniformly with `<Suspense>` boundaries.

# Detailed design

## 1. `<script async setup>` - Explicit async setup

### Syntax

```vue
<script async setup lang="ts">
const data = await fetchData()
</script>
```

The `async` keyword on the `<script setup>` tag explicitly marks the component as having an async setup phase.

### Behavior

- The component becomes an async dependency of the nearest `<Suspense>` ancestor (same as current behavior for components with top-level `await`).
- Without the `async` keyword, top-level `await` in `<script setup>` produces a **compiler error**.
- This is a compile-time check only; the runtime behavior of async setup is unchanged.

### Rationale

The `async` keyword mirrors JavaScript's own requirement that `await` can only be used inside an `async` function. This makes the async nature of the component immediately visible without reading the entire setup body.

```vue
<!-- COMPILE ERROR: top-level await requires <script async setup> -->
<script setup>
const data = await fetchData()
</script>
```

```vue
<!-- OK: async is explicit -->
<script async setup>
const data = await fetchData()
</script>
```

## 2. `v-defer` directive - Template-level promise unwrapping

### Syntax

```
v-defer="expression as identifier"
```

Where `expression` evaluates to a `Promise<T>` and `identifier` becomes a template variable of type `T` (the resolved value).

### Usage on SFC root `<template>`

When the entire component should defer its rendering:

```vue
<script setup lang="ts">
const { data } = defineProps<{ data: Promise<User> }>()
</script>

<template v-defer="data as user">
  <h1>{{ user.name }}</h1>
  <p>{{ user.email }}</p>
</template>
```

The component renders nothing until the promise resolves. It registers as an async dependency of the nearest `<Suspense>`.

### Usage on inner elements

When only part of the template should be deferred:

```vue
<script setup lang="ts">
const profile = fetchProfile()
const stats = fetchStats()
</script>

<template>
  <h1>Dashboard</h1>

  <template v-defer="profile as p">
    <ProfileCard :user="p" />
  </template>

  <template v-defer="stats as s">
    <StatsPanel :stats="s" />
  </template>
</template>
```

Each `v-defer` block independently tracks its promise. The heading renders immediately; each deferred section appears when its promise resolves.

When used on inner elements, each `v-defer` registers as a separate async dependency with the nearest `<Suspense>`. The Suspense shows its fallback until **all** deferred sections (and any async setup descendants) have resolved.

### Compilation

`v-defer` compiles to a conditional render guarded by the promise's resolution state. Conceptually:

```js
// <template v-defer="data as user"> compiles to roughly:
import { useDefer } from 'vue'

setup() {
  const { data } = defineProps(/* ... */)
  const __defer_0 = useDefer(data)
  // __defer_0 registers with nearest Suspense via provide/inject
  return { __defer_0 }
}

// render:
function render() {
  if (__defer_0.resolved) {
    const user = __defer_0.value
    return h('h1', user.name) // ...
  }
  return null // or comment node
}
```

### Promise reactivity

If the expression passed to `v-defer` is reactive (e.g., a computed that returns a new Promise), `v-defer` resets its state and re-registers with Suspense when the promise changes. The previous resolved value is discarded and the section returns to the pending state.

### Error handling

If the promise rejects, the error propagates to the nearest `<Suspense>` boundary's `onError` handler. See the Error Handling section below.

## 3. `<Suspense :with>` - Explicit promise binding

### Syntax

```vue
<Suspense :with="{ key1: promise1, key2: promise2, ... }">
  <template #default="{ key1, key2, ... }">
    <!-- key1, key2 are resolved values -->
  </template>
  <template #fallback>
    <!-- shown while any promise is pending -->
  </template>
</Suspense>
```

### Behavior

- `:with` accepts an object whose values are Promises.
- Suspense tracks all provided promises and shows the `#fallback` slot until every promise resolves.
- Resolved values are provided to the `#default` slot as scoped slot props, keyed by the same names.
- This is **additive** to existing behavior: Suspense still collects async dependencies from its subtree (async setup components and `v-defer` directives). The Suspense resolves when **all** dependencies (both `:with` promises and subtree dependencies) are settled.

### Type inference

TypeScript types flow through naturally:

```vue
<script setup lang="ts">
const user = fetchUser()   // Promise<User>
const posts = fetchPosts() // Promise<Post[]>
</script>

<template>
  <Suspense :with="{ user, posts }">
    <template #default="{ user, posts }">
      <!-- user: User, posts: Post[] -->
    </template>
  </Suspense>
</template>
```

## Error handling

All three primitives integrate with a unified error handling model on `<Suspense>`.

### `onError` event

```vue
<script setup lang="ts">
const data = fetchData() // Promise
</script>

<template>
  <Suspense
    :with="{ data }"
    @error="handleError"
  >
    <template #default="{ data }">
      <MyComponent :data="data" />
    </template>
    <template #fallback>
      <Loading />
    </template>
    <template #error="{ error, retry }">
      <p>Error: {{ error.message }}</p>
      <button @click="retry">Retry</button>
    </template>
  </Suspense>
</template>
```

- **`#error` slot**: A new named slot that renders when any tracked promise rejects. Receives `error` (the rejection reason) and `retry` (a function to re-execute all pending promises).
- **`@error` event**: Emitted when a promise rejects. Receives the error object. Useful for logging or side effects.

Error sources:
- A `:with` promise rejects
- A `v-defer` promise in the subtree rejects
- An async setup component in the subtree throws during setup

## How the three primitives compose

| Primitive | Phase | Who provides Suspense? | Promise visibility |
|---|---|---|---|
| `<script async setup>` | Setup | Parent | Implicit (opt-in via `async` keyword) |
| `v-defer` | Render | Ancestor | Explicit in child template |
| `<Suspense :with>` | Render | Self | Explicit in Suspense template |

### Composition example

A realistic example combining all three primitives:

```vue
<!-- App.vue -->
<script setup>
const config = fetchConfig() // Promise, not awaited
</script>

<template>
  <Suspense :with="{ config }">
    <template #default="{ config }">
      <Layout :config="config">
        <!-- AsyncDashboard uses <script async setup> -->
        <Suspense>
          <AsyncDashboard :theme="config.theme" />
          <template #fallback>
            <DashboardSkeleton />
          </template>
        </Suspense>
      </Layout>
    </template>
    <template #fallback>
      <AppLoader />
    </template>
  </Suspense>
</template>
```

```vue
<!-- AsyncDashboard.vue -->
<script async setup lang="ts">
const props = defineProps<{ theme: string }>()
const layout = await fetchDashboardLayout(props.theme)
const metricsPromise = fetchMetrics() // not awaited, passed down
</script>

<template>
  <div :class="theme">
    <h1>{{ layout.title }}</h1>
    <MetricsPanel :source="metricsPromise" />
  </div>
</template>
```

```vue
<!-- MetricsPanel.vue -->
<script setup lang="ts">
const { source } = defineProps<{ source: Promise<Metrics> }>()
</script>

<template v-defer="source as metrics">
  <div class="metrics">
    <StatCard v-for="stat in metrics.items" :key="stat.id" :stat="stat" />
  </div>
</template>
```

In this example:
1. `App.vue` uses `:with` to manage config loading (self-contained Suspense)
2. `AsyncDashboard.vue` uses `async setup` to fetch layout (delegated Suspense)
3. `MetricsPanel.vue` uses `v-defer` to unwrap a promise prop (component-level deferral)

## SSR considerations

### `<script async setup>`

Behavior is unchanged from the current implementation. The server waits for the async setup to complete before rendering the component's template to HTML.

### `v-defer`

On the server, `v-defer` awaits the promise and renders with the resolved value. This ensures the server-rendered HTML includes the final content, matching what the client will hydrate to.

If the promise rejects during SSR, the error propagates to the Suspense boundary or the SSR error handler.

### `<Suspense :with>`

On the server, Suspense awaits all `:with` promises before rendering the `#default` slot with the resolved values. The `#fallback` slot is never rendered on the server.

## DevTools integration

To address the debugging difficulties of current Suspense:

- Each `v-defer` directive reports its promise state (pending/resolved/rejected) and the source expression to DevTools.
- `<Suspense>` components show a list of all tracked dependencies: `:with` promises (by key name), `v-defer` instances (by source expression), and async setup components (by component name).
- The `async` keyword on `<script setup>` is reflected in the component inspector.

# Drawbacks

- **Three new/modified primitives**: This introduces meaningful API surface. However, each primitive serves a distinct purpose (setup-phase, render-phase in-component, render-phase at-boundary) and they compose cleanly.
- **Migration cost**: Existing `<script setup>` components with top-level `await` must add the `async` keyword. This is a mechanical change but is technically a breaking change.
- **`v-defer` is a new directive**: Adding a new built-in directive increases the learning surface. However, `v-defer` is only needed for the promise-as-prop pattern; the other two patterns cover the majority of use cases.
- **`v-defer` on root `<template>`**: Allowing a directive on the SFC root `<template>` tag is a new concept. However, it is syntactically natural and clearly communicates "this entire component is deferred."
- **Compiler complexity**: `v-defer` requires new compilation logic for promise tracking, variable scoping (`as` syntax), and Suspense registration.

# Alternatives

## 1. Keep implicit async detection (status quo)

Do not require `<script async setup>`. This avoids a breaking change but leaves the explicitness problem unsolved. Developers continue to be surprised by implicit Suspense dependencies.

## 2. `use()` hook instead of `v-defer` (React-style)

Provide a `use(promise)` composable that suspends the component:

```js
const result = use(fetchData())
```

This is simpler but less explicit in the template - you cannot see what is deferred by reading the template alone. It also conflates setup-phase and render-phase async into a single mechanism.

## 3. `<Await>` component instead of `v-defer`

```vue
<Await :promise="data" v-slot="result">
  <p>{{ result }}</p>
</Await>
```

This is viable and avoids adding a new directive. However:
- It introduces an extra wrapper component in the virtual DOM
- The directive form is more concise and consistent with Vue's existing directive patterns (`v-if`, `v-for`)
- It does not naturally express "defer the entire component's template"

## 4. Only `<Suspense :with>` without `v-defer`

Rely solely on Suspense-level promise management. This covers many use cases but does not support the reusable "promise-receiving component" pattern where the child handles its own unwrapping.

## 5. `<Suspense :with>` implicit shadowing (without scoped slot)

Instead of requiring `<template #default="{ config }">` to receive resolved values, the compiler could automatically shadow `:with` keys in the default slot scope. The resolved values would replace the original Promise variables by name:

```vue
<script setup lang="ts">
const config = fetchConfig() // Promise<Config>
</script>

<template>
  <Suspense :with="{ config }">
    <!-- config is automatically the resolved Config, not the Promise -->
    <Layout :config="config" />
    <template #fallback>
      <AppLoader />
    </template>
  </Suspense>
</template>
```

This is more concise and eliminates the repetitive `<template #default="{ config }">` wrapper. The compiler would:

1. Detect the keys of the `:with` object (`config`)
2. In the default slot scope, shadow those identifiers with the resolved values
3. TypeScript would infer the shadowed type as `Awaited<T>` (e.g., `Config` instead of `Promise<Config>`)

This approach is appealing for ergonomics but has trade-offs:
- **Implicit scope mutation**: The same identifier (`config`) changes type depending on whether it's inside or outside the `<Suspense>`. This can be confusing when reading the template.
- **Compiler magic**: Automatic variable shadowing without explicit syntax is a new concept in Vue templates and may surprise developers.
- **Composition with `v-if`/`v-for`**: Scope shadowing rules become complex when combined with other structural directives.
- **Explicit is better**: The scoped slot pattern `#default="{ config }"` makes it clear that `config` has been transformed, which aligns with the RFC's goal of explicitness.

This could be reconsidered as sugar syntax in a future iteration if the explicit pattern proves too verbose in practice.

## 6. `:cause` prop for annotation

The earlier draft included a `:cause` prop on `<Suspense>` for annotating the reason for suspension when using `<script async setup>`. This was dropped because:
- It serves only a documentation/debugging purpose at runtime
- DevTools integration achieves the same goal more effectively
- It adds API surface without functional benefit

# Adoption strategy

## Migration from current Suspense

1. **`<script setup>` with `await`**: Add the `async` keyword. This can be automated with a codemod that detects top-level `await` in `<script setup>` blocks.

   ```diff
   - <script setup>
   + <script async setup>
     const data = await fetchData()
   ```

2. **Existing `<Suspense>` usage**: No changes required. Existing Suspense boundaries continue to work. The `:with` prop and `v-defer` are additive features.

3. **Deprecation period**: The compiler can emit a warning (instead of an error) for top-level `await` without `async` for one minor version cycle before making it an error.

## Teaching

- **Start with `:with`**: For most data-fetching scenarios, `<Suspense :with>` is the simplest and most self-contained pattern. Teach this first.
- **Introduce `async setup` next**: For cases where the component itself needs to perform async initialization before rendering.
- **Teach `v-defer` last**: For advanced composition patterns where promises are passed through the component tree.

# Unresolved questions

1. **`v-defer` without a `<Suspense>` ancestor**: Should this be a compile-time warning, a runtime warning, or silently render nothing until resolved? The current proposal suggests a compile-time warning, but there may be valid use cases for standalone `v-defer` (e.g., progressive rendering without a loading state).

2. **Multiple `v-defer` interaction with Suspense**: When a Suspense boundary has multiple `v-defer` descendants, should the Suspense show fallback until ALL resolve, or should each `v-defer` section independently appear? The current proposal says ALL must resolve (matching existing Suspense semantics), but per-section progressive rendering could be valuable.

3. **`v-defer` and `v-if`/`v-for` interaction**: How should `v-defer` compose with other structural directives? Priority order, nesting rules, and edge cases need to be specified. A likely rule: `v-defer` cannot coexist with `v-if` or `v-for` on the same element (use a wrapping `<template>` instead).

4. **Promise identity and caching**: When a reactive source produces a new Promise (same reference or different), should `v-defer` re-suspend? The current proposal says yes (reset on new promise), but this may cause unnecessary fallback flashes. A `keepPrevious` option could be considered.

5. **Hydration mismatch**: Since the server renders the resolved state and the client starts with the pending state, there is a potential hydration mismatch window. The hydration strategy for `v-defer` needs careful specification.

6. **`<Suspense :with>` reactivity**: If a `:with` value is a ref that changes from one Promise to another, should Suspense re-suspend? How does this interact with `<Suspense>`'s existing `timeout` and `suspensible` options?
