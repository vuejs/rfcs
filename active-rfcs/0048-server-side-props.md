- Start Date: 2026-03-06
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: (leave this empty)

# Summary

Introduce two new SFC features for server-side data fetching in Vue:

1. **`defineServerSideProps`** - a compiler macro for declaring server-side data fetching logic that runs on every request during SSR.
2. **`<script server>`** - a new SFC script block that is compiled exclusively for server-side execution and stripped from the client bundle.

These features bring a co-located, per-component server-side data fetching pattern to Vue SFCs, inspired by Next.js's `getServerSideProps`, while staying idiomatic to Vue's Composition API and SFC conventions.

# Basic example

## `defineServerSideProps`

```html
<script setup>
import { computed } from 'vue'

const { repo } = defineServerSideProps(async (context) => {
  const res = await fetch('https://api.github.com/repos/vuejs/core')
  const repo = await res.json()
  return {
    props: {
      repo,
    },
  }
})

const stars = computed(() => repo.value.stargazers_count)
</script>

<template>
  <div>
    <h1>{{ repo.name }}</h1>
    <p>Stars: {{ stars }}</p>
  </div>
</template>
```

## `<script server>`

```html
<script server>
import { db } from '~/server/db'

export const fetchUser = async (id: string) => {
  return await db.user.findUnique({ where: { id } })
}
</script>

<script setup>
const { user } = defineServerSideProps(async (context) => {
  const user = await fetchUser(context.params.id)
  return {
    props: { user },
  }
})
</script>

<template>
  <div>
    <h1>{{ user.name }}</h1>
    <p>{{ user.email }}</p>
  </div>
</template>
```

## Redirect and Not Found

```html
<script setup>
const { user } = defineServerSideProps(async (context) => {
  const user = await fetchUser(context.params.id)

  if (!user) {
    return { notFound: true }
  }

  if (user.suspended) {
    return {
      redirect: {
        destination: '/suspended',
        permanent: false,
      },
    }
  }

  return {
    props: { user },
  }
})
</script>
```

# Motivation

## Problem

Currently, Vue does not have a standardized, built-in pattern for per-component server-side data fetching in SSR contexts. Frameworks like Nuxt provide their own solutions (`useAsyncData`, `useFetch`), but there is no first-party convention at the Vue SFC level. This leads to:

1. **Framework fragmentation** - Each meta-framework implements its own data fetching conventions, making it harder for the ecosystem to share components and patterns.
2. **No clear server/client boundary in SFCs** - Developers must rely on runtime checks or framework-specific conventions to ensure certain code only runs on the server.
3. **Security risks** - Without a clear compile-time server/client boundary, it is easy to accidentally ship server-only code (database credentials, API keys, internal service calls) to the client bundle.
4. **Verbose patterns** - Existing approaches require significant boilerplate to fetch data on the server, serialize it, and hydrate it on the client.

## Goals

- Provide a **first-party, framework-agnostic** SSR data fetching primitive at the SFC compiler level.
- Establish a **clear compile-time boundary** between server-only and client code within a single `.vue` file.
- Ensure server-only code (secrets, database access, internal APIs) is **never included in the client bundle**.
- Return **serializable props** that are automatically hydrated on the client, following the same mental model as `defineProps`.
- Support common SSR patterns: **redirect**, **not found**, and **request context** access.

# Detailed design

## `defineServerSideProps`

### API Signature

`defineServerSideProps` is a compiler macro (like `defineProps`, `defineEmits`) available inside `<script setup>`. It takes a single async callback function that receives a `ServerSideContext` object and returns a result object.

```ts
function defineServerSideProps<T extends Record<string, any>>(
  fn: (context: ServerSideContext) => Promise<
    | { props: T }
    | { redirect: RedirectResult }
    | { notFound: true }
  >
): ToRefs<T>
```

### `ServerSideContext`

The context object provides request-time information:

```ts
interface ServerSideContext {
  /** Route params (e.g. { id: '123' } for /user/:id) */
  params: Record<string, string>
  /** Parsed query string parameters */
  query: Record<string, string | string[]>
  /** The incoming HTTP request object (Node.js IncomingMessage) */
  req: IncomingMessage
  /** The HTTP response object (Node.js ServerResponse) */
  res: ServerResponse
  /** The resolved URL path */
  resolvedUrl: string
  /** Current locale (if i18n is configured) */
  locale?: string
  /** All configured locales */
  locales?: string[]
  /** The default locale */
  defaultLocale?: string
}
```

### Return Values

The callback must return one of three shapes:

#### 1. `{ props: T }`

Returns data to the component. The returned props must be JSON-serializable. They are available in `<script setup>` as reactive refs (via `ToRefs<T>`).

```ts
return {
  props: {
    user: { name: 'Alice', email: 'alice@example.com' },
  },
}
```

#### 2. `{ redirect: RedirectResult }`

Instructs the SSR runtime to perform a redirect instead of rendering the component.

```ts
interface RedirectResult {
  destination: string
  permanent: boolean
}
```

```ts
return {
  redirect: {
    destination: '/login',
    permanent: false, // 307 Temporary Redirect (true = 308 Permanent)
  },
}
```

#### 3. `{ notFound: true }`

Instructs the SSR runtime to render a 404 response.

```ts
return { notFound: true }
```

### Compilation Behavior

The SFC compiler handles `defineServerSideProps` as follows:

**Server build:**
The callback function is extracted and compiled into a separate server-only module export. During SSR, the framework runtime calls this function before rendering the component, passing in the request context.

**Client build:**
The callback body is entirely removed from the client bundle. The compiler replaces the `defineServerSideProps` call with a hydration helper that reads the serialized props from the SSR payload (e.g., embedded in `window.__VUE_SSR_PROPS__` or a `<script>` tag).

**Compiled output (conceptual):**

Server:
```js
// server module
export async function __serverSideProps(context) {
  const res = await fetch('https://api.github.com/repos/vuejs/core')
  const repo = await res.json()
  return { props: { repo } }
}

// component setup uses the resolved props directly
export default {
  setup(__props) {
    const repo = toRef(__props, 'repo')
    return { repo }
  }
}
```

Client:
```js
// client module - no server code present
export default {
  setup() {
    // reads pre-serialized props from SSR payload
    const __ssrProps = __hydrateServerSideProps()
    const repo = toRef(__ssrProps, 'repo')
    return { repo }
  }
}
```

### Constraints

- `defineServerSideProps` can only be used once per component, at the top level of `<script setup>`.
- The callback must be a pure async function with no references to client-side reactive state.
- Return values must be JSON-serializable. Non-serializable values (functions, symbols, class instances) will cause a compile-time or runtime error.
- `defineServerSideProps` cannot be used together with `defineProps` — the server-side props become the component's props.

## `<script server>`

### Syntax

A new SFC block type that declares server-only code:

```html
<script server>
// This code is ONLY included in the server bundle
import { db } from '~/server/db'

export const getUser = async (id) => {
  return await db.user.findUnique({ where: { id } })
}
</script>
```

### Behavior

- Code inside `<script server>` is **completely removed** from the client bundle during compilation. This is enforced at the SFC compiler level, similar to how `<style scoped>` is compiled differently.
- Exports from `<script server>` are importable by `defineServerSideProps` within the same SFC. The compiler resolves these references at compile time.
- `<script server>` can import Node.js built-in modules (`fs`, `crypto`, `path`, etc.) and server-only packages without affecting the client bundle size.
- A component may have at most one `<script server>` block.
- `<script server>` supports both JavaScript and TypeScript (via `<script server lang="ts">`).

### Compilation

**Server build:**
The `<script server>` block is compiled as a separate module that is co-located with the component. Its exports are available to the `defineServerSideProps` callback.

**Client build:**
The entire `<script server>` block is stripped. Any imports from `<script server>` used within `defineServerSideProps` are already removed along with the callback body. If `<script server>` exports are referenced outside of `defineServerSideProps` (e.g., in `<script setup>` or `<template>`), the compiler emits an error.

### Interaction with `<script setup>`

A component can have both `<script server>` and `<script setup>`:

```html
<script server lang="ts">
import { db } from '~/server/db'

export interface User {
  id: string
  name: string
  email: string
}

export const getUser = async (id: string): Promise<User | null> => {
  return await db.user.findUnique({ where: { id } })
}
</script>

<script setup lang="ts">
// Type-only imports from <script server> ARE allowed in client code
import type { User } from './MyComponent.vue?server'

const { user } = defineServerSideProps(async (context) => {
  // Runtime imports from <script server> are only available here
  const user = await getUser(context.params.id)
  if (!user) return { notFound: true }
  return { props: { user } }
})
</script>

<template>
  <div>{{ user.name }}</div>
</template>
```

**Type-only imports:** Type exports from `<script server>` can be used in `<script setup>` via `import type`. Since types are erased at compile time, this does not leak server code to the client.

## SSR Runtime Integration

The SSR runtime (provided by the meta-framework) is responsible for:

1. **Calling `__serverSideProps`** before rendering each component during SSR.
2. **Passing the `ServerSideContext`** object populated from the incoming HTTP request.
3. **Handling `redirect` and `notFound` results** by sending the appropriate HTTP response instead of rendering the component.
4. **Serializing the resolved props** into the HTML response for client-side hydration.
5. **Client-side navigation:** When the user navigates on the client side via the router, the framework should make an API request to the server to execute `__serverSideProps` and return the serialized props as JSON (similar to Next.js's behavior).

Vue core provides the compiler transforms and hydration utilities. The actual HTTP handling and routing integration is left to meta-frameworks (Nuxt, Vite SSR plugins, etc.).

## TypeScript Support

```ts
// Full type inference from the callback return type
const { repo } = defineServerSideProps(async (context) => {
  const res = await fetch('https://api.github.com/repos/vuejs/core')
  const repo: { name: string; stargazers_count: number } = await res.json()
  return { props: { repo } }
})

// repo is Ref<{ name: string; stargazers_count: number }>
repo.value.name // string
repo.value.stargazers_count // number
```

## Error Handling

If an error is thrown inside the `defineServerSideProps` callback during SSR, the framework should handle it according to its error handling conventions (e.g., render an error page, trigger `onErrorCaptured`). The specific error handling behavior is delegated to the meta-framework.

# Drawbacks

- **Increased SFC compiler complexity.** Adding `<script server>` as a new block type and `defineServerSideProps` as a new macro adds complexity to the SFC compiler and tooling (Volar, eslint-plugin-vue, etc.).
- **SSR-only feature.** This feature is only meaningful in SSR contexts. For SPA-only applications, `defineServerSideProps` has no purpose. This could confuse developers who are not working with SSR.
- **Per-request execution cost.** Unlike static generation, `defineServerSideProps` runs on every request. Developers must be aware of the performance implications and apply caching strategies where appropriate.
- **Potential overlap with existing solutions.** Nuxt already has mature data fetching primitives (`useAsyncData`, `useFetch`). Introducing a Vue-level primitive may create friction or confusion about which to use.
- **JSON serialization constraint.** All props must be JSON-serializable, which prevents passing complex objects like `Date`, `Map`, `Set`, or class instances directly. This is an inherent limitation of the SSR serialization boundary.
- **Cannot be implemented purely in user space.** This requires changes to the SFC compiler, which means it cannot be shipped as a plugin.

# Alternatives

## 1. Status quo: leave to meta-frameworks

Continue letting Nuxt and other frameworks define their own data fetching patterns. This avoids compiler complexity but perpetuates fragmentation.

## 2. Composition API utility (`useServerSideProps`)

Instead of a compiler macro, provide a runtime composable:

```ts
const { data } = useServerSideProps(async () => {
  return await fetch('/api/data').then(r => r.json())
})
```

This is simpler but cannot provide compile-time dead code elimination of server-only code. The server/client boundary would need to be enforced by convention rather than by the compiler.

## 3. Separate `.server.vue` files

Instead of `<script server>`, use a file naming convention (e.g., `MyComponent.server.vue`) where the entire component is server-only. This is less flexible as it does not allow mixing server and client code in a single file.

## 4. `defineLoader` (vue-router RFC)

The vue-router team has explored a `defineLoader` pattern that ties data loading to route definitions. This is complementary rather than competing — `defineLoader` is router-centric while `defineServerSideProps` is component-centric and SSR-focused.

# Adoption strategy

- **Non-breaking change.** This is a purely additive feature. Existing Vue applications are unaffected.
- **Opt-in.** Developers can adopt `defineServerSideProps` and `<script server>` incrementally, one component at a time.
- **Meta-framework integration.** The feature is designed so that meta-frameworks (Nuxt, Quasar, etc.) can integrate it into their existing SSR pipelines. Vue core provides the compiler transforms and hydration primitives; frameworks provide the runtime glue.
- **Tooling.** Volar and eslint-plugin-vue need to be updated to:
  - Recognize `<script server>` blocks.
  - Provide IntelliSense for `defineServerSideProps` and `ServerSideContext`.
  - Emit errors when `<script server>` exports are used outside of `defineServerSideProps` (except type-only imports).
- **Documentation.** The Vue SSR guide should be updated with a dedicated section explaining these features, including examples for common patterns (database queries, authentication checks, redirects).

# Unresolved questions

1. **Naming:** Is `defineServerSideProps` the best name? Alternatives include `defineServerProps`, `defineSSRProps`, or `defineServerData`. The current name is chosen for clarity and familiarity with the Next.js convention.
2. **Serialization strategy:** Should Vue provide a built-in serialization layer that supports `Date`, `Map`, `Set` (e.g., via `devalue` like Nuxt/SvelteKit), or stick with plain JSON?
3. **Caching:** Should the API include a built-in caching mechanism (e.g., `cache` option in the return value), or leave caching entirely to the framework/user?
4. **Streaming SSR compatibility:** How does `defineServerSideProps` interact with streaming SSR? Should props resolution block the stream, or can it be integrated with Suspense-based streaming?
5. **Nested components:** Should `defineServerSideProps` be limited to page/route-level components, or should it work in any component in the tree? Allowing it in nested components adds complexity (waterfall fetching, multiple serialization points) but is more flexible.
6. **Client-side navigation API:** The exact protocol for fetching server-side props during client-side navigation (e.g., dedicated endpoint path, request format) needs to be specified or left as a framework concern.
7. **Relationship with `defineLoader`:** How should this feature coexist with vue-router's `defineLoader` proposal? Should they share any underlying primitives?
