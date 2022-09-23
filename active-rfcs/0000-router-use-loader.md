- Start Date: 2022-07-14
- Target Major Version: Vue 3, Vue Router 4
- Reference Issues:
- Implementation PR: - https://github.com/posva/unplugin-vue-router/tree/main/src/data-fetching

# Todo List

List of things that haven't been added to the document yet:

- [x] ~~Show how to use the data loader without `vue-router/auto`~~
- [x] ~~Explain what `vue-router/auto` brings~~
- [ ] Extendable API for data fetching libraries like vue-apollo, vuefire, vue-query, etc

# Summary

There is no silver bullet to data fetching because of the different data fetching strategies and how they can define the architecture of the application and its UX. However, I think it's possible to find a solution that is flexible enough to promote good practices and reduce the complexity of data fetching in applications.
That is the goal of this RFC, to standardize and improve data fetching with vue-router:

- Automatically integrate fetching to the navigation cycle (or not by making it _lazy/non-blocking_)
- Automatically rerun when used params/query params/hash changes (avoid unnecessary fetches)
- Basic client-side caching with time-based expiration to only fetch once per navigation while using it anywhere
- Provide control over loading/error states
- Allow parallel or sequential data fetching (loaders that use each other)

This proposal concerns the Vue Router 4 but some examples concern [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) usage for improved DX. Especially the typed routes usage. Note this new API doesn't require the mentioned plugin but it **greatly improves the DX**.

# Basic example

We can define any amount of _loaders_ by **exporting them** in **page components** (components associated to a route). They return a **composable that can be used in any component** (not only pages).

A loader is **exported** from a non-setup `<script>` in a page component:

```vue
<script lang="ts">
import { getUserById } from '../api'
import { defineLoader } from 'vue-router'

// name the loader however you want **and export it**
export const useUserData = defineLoader(async (route) => {
  const user = await getUserById(route.params.id)
  // ...
  // return anything you want to expose
  return user
})

// Optional: define other component options
export default defineComponent({
  name: 'custom-name',
  inheritAttrs: false
})
</script>

<script lang="ts" setup>
// find the user as `data` and some other properties
const { data: user, pending, error, refresh } = useUserData()
// data is always present, pending changes when going from '/users/2' to '/users/3'
</script>
```

- `user`, `pending`, and `error` are refs and therefore reactive.
- `refresh` is a function that can be called to force a refresh of the data without a new navigation.
- `useUserData()` can be used in **any component**, not only in the one that defines it.
- **Only page components** can export loaders but **loaders can be defined anywhere**.
- Loaders smartly know which params/query params/hash they depend on to force a refresh when navigating:
  - Going from `/users/2` to `/users/3` will refresh the data no matter how recent the other fetch was
  - Going from `/users?name=fab` to `/users?name=fab#filters` checks if the current client side cache is recent enough to not fetch again

The simplest of loaders can be defined in just one line and types will be automatically inferred:

```ts
export const useBookCollection = defineLoader(fetchBookCollection)
// function fetchBookCollection(): Promise<Books[]>
```

Note that this syntax will intentionally be avoided in the RFC. Instead, we will often use slightly longer examples to make things easier to follow by anyone.

# Motivation

There are currently too many ways of handling data fetching with vue-router and all of them have problems:

- With navigation guards:
  - using `meta`: complex to setup even for simple cases, too low level for such a common case
  - using `onBeforeRouteUpdate()`: missing data when entering tha page
  - `beforeRouteEnter()`: non typed and non-ergonomic API with `next()`, requires a data store (pinia, vuex, apollo, etc)
  - using a watcher on `route.params...`: component renders without the data (doesn't work with SSR)

The goal of this proposal is to provide a simple yet configurable way of defining data loading in your application that is easy to understand and use. It should also be compatible with SSR and allow extensibility. It should be adoptable by frameworks like Nuxt.js to provide an augmented data fetching layer.

There are features that are out of scope for this proposal but should be implementable in user-land thanks to an _extendable API_:

- Implement a full-fledged cached API like vue-query
- Implement pagination
- Automatically refetch data when **outside of navigations** (e.g. there is no intention to implement advanced APIs such as `refetchInterval`, refetch on focus, etc)

This RFC also aims to integrate data fetching within navigations while still not forcing you to block navigation with data fetching. This pattern is useful for multiple reasons:

- Ensure data is present before mounting the component
- Enables the UX pattern of letting the browser handle loading state (aligns better with [future browser APIs](https://github.com/WICG/navigation-api))
- Makes scrolling work out of the box when navigating between pages
- Ensure fetching happens only once
- Extremely lightweight compared to more complex fetching solutions like vue-query/tastack-query, apollo/graphql, etc

# Detailed design

Ideally, this should be used alongside [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) to provide a better DX by automatically adding the proper meta fields **but it's not necessary**. At the moment, this API is implemented in that plugin as an experiment, check the [Experimental Data fetching](https://github.com/posva/unplugin-vue-router/tree/main/src/data-fetching). By default, it:

- Checks named exports in page components to set a meta property in the generated routes
- Adds a navigation guard that resolve loaders (should be moved to vue-router later)
- Implements a `defineLoader()` composable (should be moved to vue-router later)

`defineLoader()` takes a function that returns a promise (of data) and returns a composable that **can be used in any component**, not only in the one that defines it. We call these _loaders_. Loaders **must be exported by page components** in order for them to get picked up and executed during the navigation. They receive the target `route` as an argument and must return a Promise of an object of properties that will then be directly accessible **as refs** when calling the composable.

Limiting the loader access to only the target route, ensures that the data can be fetched when the user refresh the page. In enforces a good practice of correctly storing the necessary information in the route as params or query params to create sharable URLs. Within loaders there is no current instance and no access to `inject`/`provide` APIs.

Loaders also have the advantage of behaving as singleton requests. This means that they are only fetched once per navigation no matter how many page components export the loader or how many regular components use it. It also means that all the refs (`data`, `pending`, etc) are created only once, in a detached effect scope.

## Setup

To setup the loaders, we first need to setup the navigation guards:

```ts
import { setupLoaderGuard, createRouter } from 'vue-router'

const router = createRouter({
  // ...
})

setupLoaderGuard(router)
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

When using vue router named views, each named view can have their own loaders but note any navigation to the route will trigger **all loaders from all page components**.

Note: with unplugin-vue-router, a named view can be declared by appending `@name` at the end of the file name:

```text
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
  data, // Ref<T> T being the awaited return type of the function passed to `defineLoader()`
  pending, // Ref<boolean>
  error, // Ref<any>
  refresh, // () => Promise<void>
  invalidate, // () => void
  pendingLoad, // () => Promise<void> | null | undefined
} = useLoader()
```

- `refresh()` calls `invalidate()` and then invokes the loader (an internal version that sets the `pending` and other flags)
- `invalidate()` updates the cache entry time in order to force a reload next time it is triggered
- `pendingLoad()` returns a promise that resolves when the loader is done or `null` if no load is currently pending

Blocking loaders (the default) also return a ref of the returned data by the loader. Usually, it makes sense to rename this `data` property:

```ts
import { getUserById } from '@/api/users'

const useUserData = defineLoader(async ({ params }) => {
  const user = await getUserById(params.id)
  return user // can be anything
})

const {
  // ... same as above
  data: user // Ref<UserData>
} = useUserData()
```

## `setupLoaderGuard()`

`setupLoaderGuard()` setups a navigation guard that handles all the loaders. In SPA, its usage is very simple:

```ts
setupLoaderGuard(router)
```

You can also pass a second argument for some global options. In [SSR](#ssr), you can also retrieve the fetchedData as its returned value:

```ts
const fetchedData = setupLoaderGuard(
  router, // the router instance for the app
  {
    // hook triggered before each loader is ran
    async beforeLoad(route) {
      // route is the target route passed to a loader
      // Ensures pinia stores are called with the right context
      setActivePinia(pinia)

      // all loaders will await for this to be run before executing
      await someOperation
    },

    // initial data for SSR, see the section below
    initialData: {
      // ...
    },

    // returns the result of the navigation that should be returned by the navigation guard
    // see the section below for more details about "Navigation Results"
    selectNavigationResult(results) {
      return results[0]
    }
  }
)
```

## Parallel Fetching

By default, loaders are executed as soon as possible, in parallel. This scenario works well for most use cases where data fetching only requires route params/query params or nothing at all.

## Sequential fetching

Sometimes, requests depend on other fetched data (e.g. fetching additional user information). For these scenarios, we can simply import the other loaders and use them **within a different loader**:

Call **and `await`** the loader inside the one that needs it, it will only be fetched once no matter how many times it is called:

```ts
// import the loader for user information
import { useUserData } from '@/loaders/users'

export const useUserCommonFriends = defineLoader(async (route) => {
  // loaders must be awaited inside other loaders
  // .                   ⤵
  const { data: user } = await useUserData() // magically works

  // fetch other data
  const commonFriends = await getCommonFriends(user.value.id)
  return { ...user.value, commonFriends }
})
```

### Nested invalidation

Since `useUserCommonFriends()` loader calls `useUserData()`, if `useUserData()`'s cache expires or gets manually invalidated, it will also automatically invalidate `useUserCommonFriends()`.

Note that two loaders cannot use each other as that would create a _dead lock_.

This can get complex with multiple pages exposing the same loader and other pages using some of this _already exported_ loaders within other loaders. But it's not an issue, **the user shouldn't need to handle anything differently**, loaders are still only called once:

```ts
import { getFriends, getUserById } from '@/api/users'

export const useUserData = defineLoader(async (route) => {
  const user = await getUserById(route.params.id)
  return user
})

export const useCurrentUserData(async route => {
  const me = await getCurrentUser()
  // imagine legacy APIs that cannot be grouped into one single fetch
  const friends = await getFriends(user.value.id)

  return { ...me, friends }
})

export const useUserCommonFriends = defineLoader(async (route) => {
  const { data: user } = await useUserData()
  const { data: me } = await useCurrentUserData()

  const friends = await getCommonFriends(user.value.id, me.value.id)
  return { ...me.value, commonFriends: { with: user.value, friends } }
})
```

In the example above we are exporting multiple loaders but we don't need to care about the order in which they are called nor try optimizing them because **they are only called once and share the data**.

**Caveat**: must call **and await** all loaders `useUserData()` at the top of the function. You cannot put a different regular `await` in between, it has to be wrapped with a `withDataContext()`:

```ts
export const useUserCommonFriends = defineLoader(async (route) => {
  const { data: user } = await useUserData()
  await withContext(functionThatReturnsAPromise())
  const { data: me } = await useCurrentUserData()

  // ...
})
```

This is necessary to ensure nested loaders are aware of their _parent loader_. This could probably be linted with an eslint plugin. It is similar to the problem `<script setup>` had before introducing the automatic `withAsyncContext()`. The same feature could be introduced but will also have a performance cost.

## Cache and loader reuse

Each loader has its own cache and it's **not shared** across multiple application instances **as long as they use a different `router` instance** (the `router` is internally used as the key of a `WeakMap()`). This aligns with the recommendation of using one router instance per request in the ecosystem and usually one won't even need to know about this.

The cache is a very simple time based expiration cache that defaults to 5 seconds. When a loader is called, it will wait 5s before calling the loader function again. This can be changed with the `cacheTime` option:

```ts
defineLoader(..., { cacheTime: 1000 * 60 * 5 }) // 5 minutes
defineLoader(..., { cacheTime: 0 }) // No cache (still avoids calling the loader twice in parallel)
defineLoader(..., { cacheTime: Infinity }) // Cache forever
```

## Refreshing the data

When navigating, the data is refreshed **automatically based on what params, query params, and hash** are used within the loader.

Given this loader in page `/users/:id`:

```ts
export const useUserData = defineLoader(async (route) => {
  const user = await getUserById(route.params.id)
  return user
})
```

Going from `/users/1` to `/users/2` will refresh the data but going from `/users/2` to `/users/2#projects` will not unless the cache expires or is manually invalidated.

<!-- ### With navigation

TODO: maybe an option `refresh`:

```ts
export const useUserData = defineLoader(
  async (route) => {
    const friends = await getFriends(user.value.id)
    return { user, friends }
  },
  {
    // force refresh the data on navigation if /users/24?force=true
    refresh: (route) => !!route.query.force
  }
)
``` -->

### Manually refreshing the data

Manually call the `refresh()` function to force the loader to _invalidate_ its cache and _load_ again:

```vue
<script setup>
import { useInterval } from '@vueuse/core'
import { useUserData } from '~/pages/users/[id].vue'

const { data: user, refresh } = useUserData()

// refresh the data every 10s
useInterval(refresh, 10000)
</script>

<template>
  <div>
    <h1>User: {{ user.value.name }}</h1>
  </div>
</template>
```

<!--

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
  data,
  // both pending
  pending,
  // any error that happens
  error,
  // refresh both
  refresh
} = useCombinedLoader()
const { user, post } = toRefs(data.value)
</script>
```
 -->

## Usage outside of page components

Loaders can be **only exported from pages**. That's where the navigation guard picks them up, **but the page doesn't even need to use it** (invoke the composable returned by `defineLoader()`). It can be used in any component by importing the _returned composable_, even outside of the scope of the page components, by a parent.

On top of that, loaders can be **defined anywhere** and imported where using the data makes sense. This allows to define loaders in a separate `src/loaders` folder and reuse them across pages:

```ts
// src/loaders/user.ts
export const useUserData = defineLoader(...)
// ...
```

Ensure it is **exported** by page components:

```vue
<!-- src/pages/users/[id].vue -->
<script>
export { useUserData } from '~/loaders/user.ts'
</script>
```

You can still use it anywhere else:

```vue
<!-- src/components/NavBar.vue -->
<script setup>
import { useUserData } from '~/loaders/user.ts'

const { data: user } = useUserData()
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
  //                                     ^ autocompleted by unplugin-vue-router ✨
  const user = await getUserById(route.params.id)
  //                                          ^ typed!
  // ...
  return user
})
</script>

<script lang="ts" setup>
const { data: user, pending, error } = useUserData()
</script>
```

The arguments can be removed during the compilation step in production mode since they are only used for types and are ignored at runtime.

## Non blocking data fetching (Lazy Loaders)

Also known as [lazy async data in Nuxt](https://v3.nuxtjs.org/api/composables/use-async-data), loaders can be marked as lazy to **not block the navigation**.

```vue
<script lang="ts">
import { getUserById } from '../api'

export const useUserData = defineLoader(
  async (route) => {
    const user = await getUserById(route.params.id)
    return user
  },
  { lazy: true } // 👈  marked as lazy
)
</script>

<script setup>
// Differently from the example above, `user.value` can and will be initially `undefined`
const { data: user, pending, error } = useUserData()
//      ^ Ref<User | undefined>
</script>
```

This patterns is useful to avoid blocking the navigation while _less important data_ is being fetched. It will display the page earlier while some of the parts of it are still loading and you are able to display loader indicators thanks to the `pending` property.

Note this still allows for having different behavior during SSR and client side navigation, e.g.: if we want to wait for the loader during SSR but not during client side navigation:

```ts
export const useUserData = defineLoader(
  async (route) => {
    // ...
  },
  {
    lazy: !import.env.SSR, // Vite
    lazy: process.client, // NuxtJS
)
```

Existing questions:

- [~~Should it be possible to await all pending loaders with `await allPendingLoaders()`? Is it useful for SSR. Otherwise we could always ignore lazy loaders in SSR. Do we need both? Do we need to selectively await some of them?~~](https://github.com/vuejs/rfcs/discussions/460#discussioncomment-3532011)
- Should we be able to transform a loader into a lazy version of it: `const useUserDataLazy = asLazyLoader(useUserData)`

## Controlling the navigation

Since the data fetching happens within a navigation guard, it's possible to control the navigation like in regular navigation guards:

- Thrown errors (or rejected Promises) will cancel the navigation (same behavior as in a regular navigation guard) and get intercepted by [Vue Router's error handling](https://router.vuejs.org/api/interfaces/router.html#onerror)
- Redirection: `return new NavigationResult(targetLocation)` -> like `return targetLocation` in a regular navigation guard
- Cancelling the navigation: `return new NavigationResult(false)` like `return false` in a regular navigation guard

```ts
import { NavigationResult } from 'vue-router'

export const useUserData = defineLoader(
  async ({ params, path ,query, hash }) => {
    try {
      const user = await getUserById(params.id)

      return user
    } catch (error) {
      if (error.status === 404) {
        return new NavigationResult({ name: 'not-found', params: { pathMatch:  } }
        )
      } else {
        throw error // aborts the vue router navigation
      }
    }
  }
)
```

`new NavigationResult()` accepts as its only constructor argument, anything that [can be returned in a navigation guard](https://router.vuejs.org/guide/advanced/navigation-guards.html#global-before-guards).

Some alternatives:

- `createNavigationResult()`: too verbose
- `NavigationResult()` (no `new`): `NavigationResult` is not a primitive so it should use `new`

The only difference between throwing an error and returning a `NavigationResult` of an error is that the latter will still trigger the [`selectNavigationResult()` mentioned right below](#handling-multiple-navigation-results) while a thrown error will always take the priority.

### Handling multiple navigation results

Since navigation loaders can run in parallel, they can return different navigation results as well. In this case, you can decide which result should be used by providing a `selectNavigationResult()` method to `setupLoaderGuard()`:

```ts
setupLoaderGuard(router, {
  selectNavigationResult(results) {
    // results is an array of the unwrapped results passed to `new NavigationResult()`
    return results.find((result) => result.name === 'not-found')
  }
})
```

`selectNavigationResult()` will be called with an array of the unwrapped results passed to `new NavigationResult()` **after all data loaders** have been resolved. **If any of them throws an error** or if none of them return a `NavigationResult`, `selectNavigationResult()` won't be called.

## AbortSignal

The loader receives in a second argument access to an [`AbortSignal`](https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal) that can be passed on to `fetch` and other Web APIs. If the navigation is cancelled because of errors or a new navigation, the signal will abort, causing any request using it to be aborted as well.

```ts
export const useBookCatalog = defineLoader(async (_route, { signal }) => {
  const books = markRaw(await getBookCatalog({ signal }))
  return books
})
```

This aligns with the future [Navigation API](https://github.com/WICG/navigation-api#navigation-monitoring-and-interception) and other web APIs that use the `AbortSignal` to cancel an ongoing invocation.

## SSR

To support SSR we need to do two things:

- Pass a `key` to each loader so that it can be serialized into an object later. Would an array work? I don't think the order of execution is guaranteed.
- On the client side, pass the initial state to `setupLoaderGuard()`. The initial state is used once and discarded afterwards.

```ts
export const useBookCollection = defineLoader(
  async () => {
    const books = await fetchBookCollection()
    return books
  },
  { key: 'bookCollection' }
)
```

The configuration of `setupLoaderGuard()` depends on the SSR configuration, here is an example with vite-ssg:

```ts
import { ViteSSG } from 'vite-ssg'
import { setupLoaderGuard } from 'vue-router'
import App from './App.vue'
import { routes } from './routes'

export const createApp = ViteSSG(
  App,
  { routes },
  async ({ router, isClient, initialState }) => {
    // fetchedData will be populated during navigation
    const fetchedData = setupLoaderGuard(router, {
      initialData: isClient
        ? // on the client we pass the initial state
          initialState.vueRouter
        : // on server we want to generate the initial state
          undefined
    })

    // on the server, we serialize the fetchedData
    if (!isClient) {
      initialState.vueRouter = fetchedData
    }
  }
)
```

Note that `setupLoaderGuard()` **should be called before `app.use(router)`** so it takes effect on the initial navigation. Otherwise a new navigation must be triggered after the navigation guard is added.

### Avoiding double fetch on the client

One of the advantages of having an initial state is that we can avoid fetching on the client, in fact, loaders are **completely skipped** on the client if the initial state is provided. This means nested loaders **aren't executed either**. Since data loaders shouldn't contain side effects besides data fetching, this shouldn't be a problem. Note that any loader **without a key** won't be serialized and will always be executed on both client and server.

<!-- TBD: do we need to support this use case? We could allow it by having a `force` option on the loader and passing the initial state in the second argument of the loader. -->

## Performance

**When fetching large data sets**, it's convenient to mark the fetched data as _raw_ before returning it:

```ts
export const useBookCatalog = defineLoader(async () => {
  const books = markRaw(await getBookCatalog())
  return books
})
```

[More in Vue docs](https://vuejs.org/api/reactivity-advanced.html#markraw)

An alternative would be to internally use `shallowRef()` instead of `ref()` inside `defineLoader()` but that would prevent users from modifying the returned value and overall less convenient. Having to use `markRaw()` seems like a good trade off in terms of API and performance.

<!-- ## Custom `defineLoader()` for libraries

It's possible to extend the `defineLoader()` function to add new features such as a more complex cache system. TODO:

ideas:

- Export an interface that must be implemented by a composable so external libraries can implement custom strategies (e.g. vue-query)
- allow global and local config (+ types)

TODO: investigate how integrating with vue-apollo would look like -->

## Global API

It's possible to access a global state of when data loaders are fetching (during navigation or when `refresh()` is called) as well as when the data fetching navigation guard is running (only when navigating).

- `isFetchingData: Ref<boolean>`: is any loader currently fetching data? e.g. calling the `refresh()` method of a loader
- `isNavigationFetching: Ref<boolean>`: is navigation being hold by a loader? (implies `isFetchingData.value === true`). Calling the `refresh()` method of a loader doesn't change this state.

TBD: is this worth it? Are any other functions needed?

## Limitations

- Injections (`inject`/`provide`) cannot be used within a loader
- Watchers and other composables shouldn't be used within data loaders:
  - if `await` is used before calling a composable e.g. `watch()`, the scope **is not guaranteed**
  - In practice, **this shouldn't be a problem** because there is **no need** to create composables within a loader
- Global composables like pinia might need special handling (e.g. calling `setActivePinia(pinia)` in a [`beforeLoad()` hook](#setupdatafetchingguard))

# Drawbacks

- At first, it looks less intuitive than just awaiting something inside `setup()` with `<Suspense>` [but it doesn't have its limitations](#limitations)
- Requires an extra `<script>` tag but only for views. A macro `definePageLoader()`/`defineLoader()` could be error-prone as it's very tempting to use reactive state declared within the component's `<script setup>` but that's not possible as the loader must be created outside of its `setup()` function

# Alternatives

## Suspense

Using Suspense is probably the first alternative that comes to mind and it has been considered as a solution for data fetching by implementing proofs of concepts. It however suffer from major drawbacks that are tied to its current design and is not a viable solution for data fetching.

One could imagine being able to write something like:

```vue
<!-- src/pages/users.vue = /users -->
<!-- Displays a list of all users -->
<script setup>
const userList = shallowRef(await fetchUserList())

// manually expose a refresh function to be called whenever needed
function refresh() {
  userList.value = await fetchUserList()
}
</script>
```

Or when params are involved in the data fetching:

```vue
<!-- src/pages/users.[id].vue = /users/:id -->
<!-- Displays a list of all users -->
<script setup>
const route = useRoute()
const user = shallowRef(await fetchUserData(route.params.id))

// manually expose a refresh function to be called whenever needed
function refresh() {
  user.value = await fetchUserData(route.params.id)
}

// hook into navigation instead of a watcher because we want to block the navigation
onBeforeRouteUpdate(async (to) => {
  // note how we need to use `to` and not `route` here
  user.value = await fetchUserData(to.params.id)
})
</script>
```

One of the reasons to block the navigation while fetching is to align with the upcoming [Navigation API](https://github.com/WICG/navigation-api) which will show a spinning indicator (same as when entering a URL) on the browser UI while the navigation is blocked.

This setup has many limitations:

- Nested routes will force in some navigations a **sequential data fetching**: it's not possible to ensure an **optimal parallel fetching** in all cases
- Manual data refreshing is necessary **unless you add a `key` attribute** to the `<RouterView>` which will force a remount of the component on navigation. This is not ideal because it will remount the component on every navigation, even when the data is the same. It's necessary if you want to do a `<transition>` but less flexible than the proposed solution which can also use a `key` if needed.
- By putting the fetching logic within the `setup()` of the component we face other issues:

  - No abstraction of the fetching logic => **code duplication** when fetching the same data in multiple components
  - No native way to gather the same data fetching among multiple components using them: it requires using a store and extra logic to skip redundant fetches (see bottom of [Nested Invalidation](#nested-invalidation) )
  - Requires mounting the upcoming page component (while the navigation is still blocked) which can be **expensive in terms of rendering** as we still need to render the old page while we _try to mount the new page_.

- No native way of caching data, even for very simple cases (e.g. no refetching when fast traveling back and forward through browser UI)
- Not possible to precisely read (or write) the loading state (see [vuejs/core#1347](https://github.com/vuejs/core/issues/1347)])

On top of this it's important to note that this RFC doesn't limit you: you can still use Suspense for data fetching or even use both, **this API is completely tree shakable** and doesn't add any runtime overhead if you don't use it.

## Other alternatives

- Allowing blocking data loaders to return objects of properties:

  ```ts
  export const useUserData = defineLoader(async (route) => {
    const user = await getUserById(route.params.id)
    // instead of return user
    return { user }
  })
  // instead of const { data: user } = useUserData()
  const { user } = useUserData()
  ```

  This was the initial proposal but since this is not possible with lazy loaders it was more complex and less intuitive. Having one single version is overall easier to handle.

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

- Pass route properties instead of the whole `route` object:

  ```ts
  import { getUserById } from '../api'

  export const useUserData = defineLoader(async ({ params, query, hash }) => {
    const user = await getUserById(params.id)
    return { user }
  })
  ```

  This has the problem of not being able to use the `route.name` to determine the correct typed params (with unplugin-vue-router):

  ```ts
  import { getUserById } from '../api'

  export const useUserData = defineLoader(async (route) => {
    if (route.name === 'user-details') {
      const user = await getUserById(params.id)
      //                                    ^ Typed!
      return { user }
    }
  })
  ```

## Naming

Variables could be named differently and proposals are welcome:

- `pending` (same as Nuxt) -> `isPending`, `isLoading`
- Rename `defineLoader()` to `defineDataFetching()` (or others)

## Nested Invalidation syntax drawbacks

- Allowing `await getUserById()` could make people think they should also await inside `<script setup>` and that would be a problem because it would force them to use `<Suspense>` when they don't need to.

- Another alternative is to pass an array of loaders to the loader that needs them and let it retrieve them through an argument, but it feels considerably less ergonomic:

  ```ts
  import { useUserData } from '~/pages/users/[id].vue'

  export const useUserFriends = defineLoader(
    async (route, { loaders: [userData] }) => {
      const friends = await getFriends(user.value.id)
      return { ...userData.value, friends }
    },
    {
      // explicit dependencies
      waitFor: [useUserData]
    }
  )
  ```

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
    return user
  },
  // block the navigation for 1 second and then let the navigation go through
  { lazy: 1000 }
)
</script>

<script setup>
const { data, pending, error } = useUserData()
//      ^ Ref<User | undefined>
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

- Should there by an `afterLoad()` hook, similar to `beforeLoad()`?
- Is `useNuxtApp()` usable within loaders?
- Is there anything needed besides the `route` inside loaders?
- Add option for placeholder data?
- What other operations might be necessary for users?
- Is there a way to efficiently parse the exported properties in pages to filter out pages that have named exports but no loaders?
