- Start Date: 2022-02-11
- Target Major Version: Vue Router >=4
- Reference Issues:
- Implementation PR:

# Summary

Better integrating Data Fetching logic with the router navigation:

- Automatically include `await` calls within an ongoing navigation for components that are mounted as a result of the navigation
- Navigation guards declared **within** `setup()`
- Loading and Error Boundaries thanks to Suspense
- Single Fetching on Hydration (avoid fetching on server and client when doing SSR)

# Basic example

- Fetching data once

Data fetching when entering the router. This is simpler than the `onBeforeNavigate` example and is what most users would expect to work. It also integrates with the navigation:

```vue
<script setup lang="ts">
import { useUserStore } from '~/stores/user' // store with pinia

const userStore = useUserStore()
await userStore.fetchUser(to.params.id)
</script>

<template>
  <h2>{{ userStore.userInfo.name }}</h2>
  <ul>
    <li>Email: {{ userStore.userInfo.email }}</li>
    ...
  </ul>
</template>
```

- Fetching data with each navigation

Data fetching integrates with the router navigation within any component.

```vue
<script setup lang="ts">
import { onBeforeNavigate } from 'vue-router'
import { useUserStore } from '~/stores/user' // store with pinia

const userStore = useUserStore()

// Triggers on entering or updating the route:
// / -> /users/2 ✅
// /users/2 -> /users/3 ✅
// /users/2 -> / ❌
onBeforeNavigate(async (to, from) => {
  // could also be return userStore.fetchUser(...)
  await userStore.fetchUser(to.params.id)
})
</script>

<template>
  <h2>{{ userStore.userInfo.name }}</h2>
  <ul>
    <li>Email: {{ userStore.userInfo.email }}</li>
    ...
  </ul>
</template>
```

# Motivation

Today's data fetching with Vue Router is simple when the data fetching is simple enough but it gets complicated to implement in advanced use cases that involve SSR. It also doesn't work with `async setup()` + Suspense.

# Detailed design

## Fetching data once

You can do initial fetching (when it only needs to happen once) **without using anything specific to Vue Router**:

```vue
<script setup lang="ts">
import { getUserList } from '~/apis/user'

const userList = await getUserList()
// or if you need a reactive version
const userList = reactive(await getUserList())
</script>
```

- The router navigation won't be considered settled until all `await` (or returned promises inside `setup()`) are resolved
- This creates a redundant fetch with SSR because `userList` is never serialized into the page for hydration. In other words: **don't do this if you are doing SSR**.
- Simplest data fetching pattern, some people are probably already using this pattern.
- Rejected promises are caught by `router.onError()` and cancel the navigation. Note that differently from `onErrorCaptured()`, it's not possible to return `false` to stop propagating the error.

## Fetching with each navigation

Use this when the fetching depends on the route (params, query, ...) like a route `/users/:id` that allows navigating through users. All the rules above apply:

```vue
<script setup lang="ts">
import { onBeforeNavigate } from 'vue-router'
import { getUser } from '~/apis/user'

const user = ref<User>()
onBeforeNavigate(async (to) => {
  user.value = await getUser(to.params.id)
})
</script>
```

- Any component that is rendered by RouterView or one of its children can call `onBeforeNavigate()`
- `onBeforeNavigate()` triggers on entering and, updating. It **doesn't trigger when leaving**. This is because it's mostly used to do data fetching and you don't want to fetch when leaving. You can still use `onBeforeRouteLeave()`.
- Can be called multiple times
- It returns a promise that resolves when the fetching is done. This allows to await it to use any value that is updated within the navigation guard:

  ```ts
  const user = ref<User>()
  await onBeforeNavigate(async (to) => {
    user.value = await getUser(to.params.id)
  })
  user.value // populated because we awaited
  ```

- Awaiting or not `onBeforeNavigate()` doesn't change the fact that the navigation is settled only when all `await` are resolved.

## SSR support

To properly handle SSR we need to:

- Serialize the fetched data to hydrate: **which is not handled by the router**
- Avoid the double fetching issue by only fetching on the server

Serializing the state is out of scope for the router. Nuxt defines a `useState()` composable. A pinia store can also be used to store the data:

```vue
<script setup lang="ts">
import { onBeforeNavigate } from 'vue-router'
import { storeToRefs } from 'pinia'
import { useUserStore } from '~/stores/user' // store with pinia

const userStore = useUserStore()
const { userInfo } = storeToRefs(userStore)

// The navigation guard will be skipped during the first rendering on client side because everything was done on the server
// This will avoid the double fetching issue as well and ensure the navigation is consistent
await onBeforeNavigate(async (to, from) => {
  // we could return a promise too
  await userStore.fetchUser(to.params.id)

  // here we could also return false, "/login", throw new Error()
  // like a regular navigation guard
})
</script>

<template>
  <h2>{{ userInfo.name }}</h2>
  <ul>
    <li>Email: {{ userInfo.email }}</li>
    ...
  </ul>
</template>
```

To avoid the double fetching issue, we can detect the hydration during the initial navigation and skip navigation guards altogether as they were already executed on the server. **This is however a breaking change**, so it would require a new option to `createRouter()` that disables the feature by default:

```ts
createRouter({
  // ...
  skipInitialNavigationGuards: true
})
```

Another solution is to only skip `onBeforeNavigate()` guards during the initial navigation on the client if it was hydrated. This is not a breaking change since the API is new but it does make things inconsistent.

### Using a new custom `useDataFetching()`

Maybe we could expose a utility to handle SSR **without a store** but I think [serialize-revive](https://github.com/kiaking/vue-serialize-revive) should come first. Maybe internally they could use some kind of `onSerialization(key, fn)`/`onHydration(key, fn)`.

```vue
<script setup lang="ts">
import { onBeforeNavigate, useDataFetching } from 'vue-router'
import { getUserInfo } from '~/apis/user'

// only necessary with SSR without Pinia (or any other store)
const userInfo = useDataFetching(getUserInfo)
// this await allows for Suspense to wait for the _entering_ navigation
await onBeforeNavigate(async (to, from) => {
  // await (we could return a promise too)
  await userInfo.fetch(to.params.id)
})
</script>

<template>
  <h2>{{ userInfo.name }}</h2>
  <ul>
    <li>Email: {{ userInfo.email }}</li>
    ...
  </ul>
</template>
```

`useDataFetching()` is about being able to hydrate the information but `onBeforeNavigate()` only needs to return a promise:

## Canceled Navigations

A canceled navigation is a navigation that is canceled by the code (e.g. unauthorized access to an admin page) and results in the route **not changing**. It triggers `router.afterEach()`. Note this doesn't include redirecting (e.g. `return "/login"` but not `redirect: '/login'` (which is a _"rewrite"_ of the ongoing navigation)) as they trigger a whole new navigation that gets _"appended"_ to the ongoing navigation.

## Failed Navigations

An failed navigation is different from a canceled navigation because it comes from an unexpected uncaught thrown error. It also triggers `router.onError()`.

# Drawbacks

## Larger CPU/Memory usage

In order to execute any `await` statement, we have to mount the pending routes within a Suspense boundary while still displaying the current view which means two entire views are rendered during the navigation. This can be slow for big applications but no real testing has been done to prove this could have an impact on UX.

## Using Suspense

Currently Suspense allows displaying a `fallback` content after a certain timeout. This is useful when displaying a loading screen but there are other ways of telling the user we are waiting for asynchronous operations in the back like a progress bar or a global loader.
The issue with the `fallback` slot is it is **only controlled by `await` inside of mounting component**. Meaning there is no programmatic API to _reenter_ the loading state and display the `fallback` slot in this scenario:

Given a Page Component for `/users/:id`:

```vue
<script setup lang="ts">
import { onBeforeNavigate } from 'vue-router'
import { useUserStore } from '~/stores/user' // store with pinia

const userStore = useUserStore()

await onBeforeNavigate(async (to, from) => {
  await userStore.fetchUser(to.params.id)
})
</script>

<template>
  User: {{ userStore.user.name }}
  <ul>
    <li v-for="user in userStore.user.friends">
      <RouterLink :to="`/users/${user.id}`">{{ user.name }}</RouterLink>
    </li>
  </ul>
</template>
```

And an `App.vue` as follows

```vue
<template>
  <!-- SuspendedRouterView internally uses Suspense around the rendered page and inherits all of its props and slots -->
  <SuspendedRouterView :timeout="800">
    <template #fallback> Loading... </template>
  </SuspendedRouterView>
</template>
```

- Going from any other page to `/users/1` displays the "Loading..." message if the request for the user information takes more than 800ms
- Clicking on any of the users and effectively navigating to `/users/2`

# Alternatives

What other designs have been considered? What is the impact of not doing this?

- Not using Suspense: Implementing vue router's own fallback/error mechanism: not sure if doable because we need to try mounting the components to "collect" their navigation guards.

## To Suspense `fallback` not displaying on the same route navigations

We could retrieve the status of the closest `<Suspense>` boundary:

```js
const { state } = useSuspense()
state // TBD: Ref<'fallback' | 'settled' | 'error'>
```

There could maybe also be a way to programmatically change the state of the closest Suspense boundary:

```js
const { appendJob } = useSuspense()
// add a promise, enters the loading state if Suspense is not loading
appendJob(
  (async () => {
    // some async operation
  })()
)
```

## Suspense Limitations

- Any provided/computed value is updated even in the non active branches of Suspense. Would there be a way to ensure `computed` do not update (BTW keep alive also do this) or to freeze provided reactive values?

# Adoption strategy

If we implement this proposal, how will existing Vue developers adopt it? Is
this a breaking change? Can we write a codemod? Can we provide a runtime adapter library for the original API it replaces? How will this affect other projects in the Vue ecosystem?

# Unresolved questions

- What should happen when an error is thrown (See https://github.com/vuejs/core/issues/1347)
  - Currently the `fallback` slot stays visible
  - Should the router rollback the previous route location?
- `useDataFetching()`:

  - Do we need anything else apart from `data`, and `fetch()`?
    - loading/error state: even though it can be handled by Suspense `fallback` slot, we could still make them appear here for users not using a `fallback` slot

- Maybe we should completely skip the navigation guards when hydrating
