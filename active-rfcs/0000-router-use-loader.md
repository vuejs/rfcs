- Start Date: 2022-07-14
- Target Major Version: Vue 3, Vue Router 4
- Reference Issues:
- Implementation PR:

# Todo List

List of things that haven't been added to the document yet:

- [x] Show how to use the data loader without ~~`@vue-router`~~ `vue-router/auto`
- [ ] Explain what ~~`@vue-router`~~ `vue-router/auto` brings

# Summary

Standarize and improve data fetching by adding helpers to be called within page components:

- Automatically integrate fetching to the navigation cycle (or not by making it non-blocking)
- Automatically rerun when used params changes (avoid unnecessary fetches)
- Basic client-side caching with time-based expiration to only fetch once per navigation while using it anywhere
- Provide control over loading/error states
- Allow parallel or sequential data fetching (loaders that use each other)

This proposal concerns the Vue Router 4 but some examples concern [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) usage for improved DX. Especially the typed routes usage.

# Basic example

We can define any amount of loaders by **exporting them** in **page components**. They return a **composable that can be used anywhere in the app**.

```vue
<script lang="ts">
import { getUserById } from '../api'
import { defineLoader } from 'vue-router'

// name the loader however you want **and export it**
export const useUserData = defineLoader(async (route) => {
  const user = await getUserById(route.params.id)
  // ...
  // return anything you want to expose
  return { user }
})

// Optional: define other component options
export default defineComponent({
  name: 'custom-name',
  inheritAttrs: false
})
</script>

<script lang="ts" setup>
// find `user` and some other properties
const { user, pending, error, refresh } = useUserData()
// user is always present, pending changes when going from '/users/2' to '/users/3'
</script>
```

- `user`, `pending`, and `error` are refs and therefore reactive.
- `refresh` is a function that can be called to force a refresh of the data without a new navigation.
- `useUserData()` can be used in **any component**, not only in the one that defines it.
- **Only page components** can export loaders but loaders can be defined anywhere.
- Loaders smartly know which params/query params they depend on to force a refresh when navigating:
  - Going `/users/2` to `/users/3` will refresh the data no matter how recent the other fetch was
  - Going from `/users?name=fab` to `/users?name=fab#filters` checks if the current client side cache is recent enough to not fetch again

# Motivation

There are currently too many ways of handling data fetching with vue-router and all of them have problems:

- With navigation guards:
  - using `meta`: complex to setup even for simple cases, too low level for such a common case
  - using `onBeforeRouteUpdate()`: missing data when entering tha page
  - `beforeRouteEnter()`: non typed and non-ergonomic API with `next()`, requires a data store (pinia, vuex, apollo, etc)
  - using a watcher on `route.params...`: component renders without the data (doesn't work with SSR)

The goal of this proposal is to provide a simple yet configurable way of defining data loading in your application that is easy to understand and use. It should also be compatible with SSR and allow extensibility.

There are features that are out of scope for this proposal but should be implementable in user land thanks to an _extendable API_:

- Implement a full-fledged cached API like vue-query
- Implement pagination
- Automatically refetch data when **outside of navigations** (e.g. no `refetchInterval`, no refetch on focus, etc)

# Detailed design

Ideally, this should be used alongside [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) to provide a better DX by automatically adding the loaders where they should **but it's not necessary**. By default, it:

- Checks named exports in page components to set a meta property in the generated routes
- Adds a navigation guard that resolve loaders
- Implements a `defineLoader()` composable (should be moved to vue-router later)

`defineLoader()` takes a function that returns a promise (of data) and returns a composable that **can be used in any component**, not only in the one that defines it. We call these _loaders_. Loaders **must be exported by page components** in order for them to get picked up and executed during the navigation. They receive the target `route` as an argument and must return a Promise of an object of properties that will then be directly accessible **as refs** when accessing

Limiting the loader access to only the target route, ensures that the data can be fetched when the user refresh the page. In enforces a good practice of correctly storing the necessary information in the route as params or query params. Within loaders there is no current instance and no access to `inject`/`provide` APIs.

Loaders also have the advantage of behaving as singleton requests. This means that they are only fetched once per navigation no matter how many page components export the loader or how many regular components use it.

## Setup

To setup the loaders, we first need to setup the navigation guards:

```ts
import { setupDataFetchingGuard, createRouter } from 'vue-router'

const router = createRouter({
  // ...
})

setupDataFetchingGuard(router)
```

Then, for each page exporting a loader, we need to add a meta property to the route:

```ts
import { LoaderSymbol } from 'vue-router'

const routes = [
  {
    path: '/users/:id',
    component: () => import('@/pages/UserDetails.vue'),
    meta: {
      // 👇            v array of all the necessary loaders
      [LoaderSymbol]: [() => import('@/pages/UserDetails.vue')]
    },
  },
  // Named views must include all page component lazy imports
  {
    path: '/users/:id',
    components: {
     default () => import('@/pages/UserDetails.vue'),
     aux () => import('@/pages/UserDetailsAux.vue'),
    },
    meta: {
      [LoaderSymbol]: [() => import('@/pages/UserDetails.vue'), () => import('@/pages/UserDetailsAux.vue')],
    },
  },
  // Nested routes follow the same pattern, declare an array of lazy imports relevant to each routing level
  {
    path: '/users/:id',
    component: () => import('@/pages/UserDetails.vue'),
    meta: {
      [LoaderSymbol]: [() => import('@/pages/UserDetails.vue')],
    },
    children: [
      {
        path: 'edit',
        component: () => import('@/pages/UserEdit.vue'),
        meta: {
          [LoaderSymbol]: [() => import('@/pages/UserEdit.vue')],
        },
      }
    ]
  },
]
```

This is **pretty verbose** and that's why it is recommended to use [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) to make this **completely automatic**: the plugin generates the routes with the symbols and loaders. It will also setup the navigation guard when creating the router instance.
When using the plugin, any page component with **named exports will be marked** with a symbol to pick up any possible loader in a navigation guard. The navigation guard checks every named export for loaders and _load_ them.

When using vue router named views, each named view can have their own loader but note any navigation to the route will trigger **all loaders from all page components**.

Note: with unplugin-vue-router, a named view can be declared by appending `@name` at the end of the file name:

```
src/pages/
└── users/
    ├── index.vue
    └── index@aux.vue
```

This creates a `components: { default: ..., aux: ... }` entry in the route config.

## `defineLoader()`

`defineLoader()` returns a composable with the following properties:

```ts
const useLoader = defineLoader(...)
const {
  pending, // Ref<boolean>
  error, // Ref<any>
  refresh, // () => Promise<void>
  invalidate, // () => void
  pendingLoad, // () => Promise<void> | null | undefined
} = useLoader()
```

- `refresh()` calls `invalidate()` and then invokes the loader (an internal version that sets the `pending` flag and others)
- `invalidate()` sets the cache entry time to 0 to force a reload next time it has to
- `pendingLoad()` returns a promise that resolves when the loader is done or null if it no load is pending

Blocking loaders (the default) also return an objects of refs of whatever is returned by the loader.

```ts
const useLoader = defineLoader(async ({ params }) => {
  const user = await getUser(params.id)
  return { user }
})

const {
  // ... same as above
  user // Ref<UserData>
} = useLoader()
```

On the other hand, [Lazy Loaders](#non-blocking-data-fetching) return a `data` property instead:

```ts
const useLoader = defineLoader(
  async ({ params }) => {
    const user = await getUser(params.id)
    return { user }
  },
  // 👇 this is the only difference
  { lazy: true }
)

const {
  // ... same as above
  data // Ref<{ user: UserData }>
} = useLoader()
```

Note you can also just return the user directly:

```ts
const useLoader = defineLoader(
  async ({ params }) => {
    const user = await getUser(params.id)
    return user
  },
  { lazy: true }
)

const {
  // ... same as above
  data: user // Ref<UserData>
} = useLoader()
```

## Parallel Fetching

By default, loaders are executed as soon as possible, in parallel. This scenario works well for most use cases where data fetching only requires params or nothing at all.

## Sequential fetching

Sometimes, requests depend on other fetched data (e.g. fetching additional user information). For these scenarios, we can simply import the other loaders and use them **within a different loader**:

Call the loader inside the one that needs it, it will only be fetched once

```ts
export const useUserFriends = defineLoader(async (route) => {
  const { user, isLoaded } = useUserData()
  await isLoaded()
  const friends = await getFriends(user.value.id)
  return { user, friends }
})
```

Note that two loaders cannot use each other as that would create a _dead lock_.

Alternatives:

- Allowing `await getUserById()` could make people think they should also await inside `<script setup>` and that would be a problem because it would force them to use `<Suspense>` when they don't need to.

- Pass an array of loaders to the loader that needs them and use let it retrieve them through an argument:

  ```ts
  import { useUserData } from '~/pages/users/[id].vue'

  export const useUserFriends = defineLoader(
    async (route, [userData]) => {
      const friends = await getFriends(user.value.id)
      return { user, friends }
    },
    {
      waitFor: [useUserData]
    }
  )
  ```

## Cache and loader reuse

Each loader has its own cache and it's **not shared** across multiple application instances **as long as they use a different `router` instance**. This aligns with the recommendation of using one router instance per

TODO: expiration time of cache

## Refreshing the data

The data is refreshed automatically based on what params and query params are used within

### With navigation

TODO: maybe an option `refresh`:

```ts
export const useUserData = defineLoader(
  async (route) => {
    await isLoaded()
    const friends = await getFriends(user.value.id)
    return { user, friends }
  },
  {
    // force refresh the data on navigation if /users/24?force=true
    refresh: (route) => !!route.query.force
  }
)
```

### Manually

Manually call the `refresh()` function to force the loader to _invalidate_ its cache and _load_ again:

```ts
const { user, refresh } = useUserData()
```

## Combining loaders

TBD: is this necessary? At the end, this is achievable with similar composables like [logicAnd](https://vueuse.org/math/logicAnd/).

It's possible to combine multiple loaders into one loader with `combineLoaders()` to not combine the results of multiple loaders into a single object but also merge `pending`, `error`, etc

```vue
<script>
import { useUserData } from '@/pages/users/[id].vue'
import { usePostData } from '@/pages/posts/[id].vue'

export const useCombinedLoader = combineLoaders(useUserData, usePostData)
</script>

<script setup>
const {
  user,
  post,
  // both pending
  pending,
  // any error that happens
  error,
  // refresh both
  refresh
} = useCombinedLoader()
</script>
```

## Usage outside of page components

Loaders can be **only exported from pages**. That's where the navigation guard picks them up, but the page doesn't even need to use it. It can be used in any component by importing the function, even outside of the scope of the page components, by a parent.

However, loaders can be **defined anywhere** and imported where the using the data makes most sense. This allows to define loaders anywhere or even reuse loaders from a different page:

```ts
// src/loaders/user.ts
export const useUserData = defineLoader()
// ...
```

Ensure it is **exported** by page components:

```vue
<!-- src/pages/users/[id].vue -->
<script>
export { useUserData } from '~/loaders/user.ts'
</script>
```

You can still use it anywhere else

```vue
<!-- src/components/NavBar.vue -->
<script setup>
import { useUserData } from '~/loaders/user.ts'

const { user } = useUserData()
</script>
```

In such scenarios, it makes more sense to move the loader to a separate file to ensure a more concrete code splitting.

## TypeScript

Types are automatically generated for the routes by [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) and can be referenced with the name of each route to hint `defineLoader()` the possible values of the current types:

```vue
<script lang="ts">
import { getUserById } from '../api'
import { defineLoader } from 'vue-router'

export const useUserData = defineLoader('/users/[id]', async (route) => {
  //                                     ^ autocompleted
  const user = await getUserById(route.params.id)
  //                                          ^ typed!
  // ...
  return { user }
})
</script>

<script lang="ts" setup>
const { user, pending, error } = useUserData()
</script>
```

The arguments can be removed during the compilation step in production mode since they are only used for types and are actually ignored at runtime.

## Testing pages

TODO: How this affects testing

## Non blocking data fetching

Also known as [lazy async data in Nuxt](https://v3.nuxtjs.org/api/composables/use-async-data), loaders can be marked as lazy to **not block the navigation**.

```vue
<script lang="ts">
import { getUserById } from '../api'

export const useUserData = defineLoader(
  async (route) => {
    const user = await getUserById(route.params.id)
    return { user }
  },
  { lazy: true } // 👈  marked as lazy
)
</script>

<script setup>
// in this scenario, we no longer have a `user` property since `useUserData()` returns synchronously before the loader is resolved
const { data, pending, error } = useUserData()
//      ^ Ref<{ user: User } | undefined>
</script>
```

See alternatives for a version of `lazy` that accepts a number/function.

## Error handling

If a request fails, the user can catch the error in the loader and return it differently

TODO: expand

### AbortSignal

The loader receives in a second argument access to an [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) that can be passed on to `fetch` and other Web APIs. If the navigation is cancelled because of errors or a new navigation, the signal will abort, causing any request using it to be aborted.

```ts
export const useBookCatalog = defineLoader(async (_route, { signal }) => {
  const books = markRaw(await getBookCatalog({ signal }))
  return { books }
})
```

## SSR

TODO: probably need an API to allow using something else than a simple `ref()` so the data can be written anywhere and let user serialize it.

### Avoiding double fetch on the client

TODO: an API to set the initial value of the cache before mounting the app.

## Performance

When fetching large data sets, it's convenient to mark the fetched data as _raw_ before returning it:

```ts
export const useBookCatalog = defineLoader(async () => {
  const books = markRaw(await getBookCatalog())
  return { books }
})
```

[More in Vue docs](https://vuejs.org/api/reactivity-advanced.html#markraw)

An alternative would be to internally use `shallowRef()` instead of `ref()` inside `defineLoader()` but that would prevent users from modifying the returned value and overall less convenient. Having to use `markRaw()` seems like a good trade off in terms of API and performance.

## HMR

When changing the `<script>` the old cache is transferred and refreshed. Worst case, the page reloads for non lazy loaders.

TODO: expand

## Extending `defineLoader()`

It's possible to extend the `defineLoader()` function to add new features such as a more complex cache system. TODO:

ideas:

- Export an interface that must be implemented by a composable so external libraries can implement custom strategies (e.g. vue-query)
- allow global and local config (+ types)

TODO: investigate how integrating with vue-apollo would look like

## Global API

It's possible to access a global state of when data loaders are fetching (navigation + calling `refresh()`) as well as when the data fetching navigation guard is running (only when navigating).

- `isFetchingData`
- `isNavigationFetching`

## Limitations

- Injections (`inject`/`provide`) cannot be used within a loader
- Watchers and other composables can be used but **will be created in a detached effectScope** and therefore never be released. **This is why, using composables should be avoided**.

# Drawbacks

This solution is not a silver bullet but I don't think one exists because of the different data fetching strategies and how they can define the architecture of the application.

- Less intuitive than just awaiting something inside `setup()`
- Requires an extra `<script>` tag but only for views

# Alternatives

- Allowing a `before` and `after` hook to allow changing data after each loader call. e.g. By default the data is preserved while a new one is being fetched
- Should we return directly the necessary data instead of wrapping it with an object and always name it `data`?:

  ```vue
  <script lang="ts">
  import { getUserById } from '../api'

  export const useUserData = defineLoader(async (route) => {
    const user = await getUserById(route.params.id)
    return user
  })
  </script>

  <script setup>
  const { data: user, pending, error } = useUserData()
  </script>
  ```

- Adding a new `<script loader>` similar to `<script setup>`:

  ```vue
  <script lang="ts" loader="useUserData">
  import { getUserById } from '@/api/users'
  import { useRoute } from 'vue-router' // could be automatically imported

  const route = useRoute()
  // any variable created here is available in useLoader()
  const user = await getUserById(route.params.id)
  </>

  <script lang="ts" setup>
  const { user, pending, error } = useUserData()
  </>
  ```

  Is exposing every variable a good idea?

- Using Suspense to natively handle `await` within `setup()`. [See other RFC](#TODO).

## Naming

Variables could be named differently and proposals are welcome:

- `pending` (same as Nuxt) -> `isPending`, `isLoading`
- Rename `defineLoader()` to `defineDataFetching()` (or others)

## Advanced `lazy`

The `lazy` flag could be extended to also accept a number or a function. I think this is too much and should therefore not be included.

A lazy loader **always reset the data fields** to `null` when the navigation starts before fetching the data. This is to avoid blocking for a moment (giving the impression data is loading), then showing the old data while the new data is still being fetched and eventually replacing the one being shown to make it even more confusing for the end user.

Alternatively, you can pass a _number_ to `lazy` to block the navigation for that number of milliseconds:

```vue
<script lang="ts">
import { getUserById } from '../api'

export const useUserData = defineLoader(
  async (route) => {
    const user = await getUserById(route.params.id)
    return { user }
  },
  // block the navigation for 1 second and then let the navigation go through
  { lazy: 1000 }
)
</script>

<script setup>
// in this scenario, we no longer have a `user` property since `useUserData()` returns synchronously before the loader is resolved
const { data, pending, error } = useUserData()
//      ^ Ref<{ user: User } | undefined>
</script>
```

Note that lazy loaders can only control their own blocking mechanism. They can't control the blocking of other loaders. If multiple loaders are being used and one of them is blocking, the navigation will be blocked until all of the blocking loaders are resolved.

TBD: Conditionally block upon navigation / refresh:

```ts
export const useUserData = defineLoader(
  loader,
  // ...
  {
    lazy: (route) => {
      // ...
      return true // or a number
    }
  }
)
```

# Adoption strategy

Introduce this as part of [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) to test it first and make it part of the router later on.

# Unresolved questions

- Should there be a way to handle server only loaders?
- Is `useNuxtApp()` usable within loaders?
- Is there anything needed besides the `route` inside loaders?
- Add option for placeholder data?
- What other operations might be necessary for users?
