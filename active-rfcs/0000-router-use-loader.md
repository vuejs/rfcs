- Start Date: 2022-07-14
- Target Major Version: Vue 3, Vue Router 4
- Reference Issues:
- Implementation PR:

# Summary

Standarize and improve data fetching by adding helpers to be called within page components:

- Automatically integrate fetching to the navigation cycle (or not by making it non-blocking)
- Automatically rerun when used params changes (avoid unnecessary fetches)
- Provide control over loading/error states
- Allow parallel or sequential data fetching (loaders that use each other)

This proposal concerns the Vue Router 4 with [unplugin-vue-router](https://github.com/posva/unplugin-vue-router)

# Basic example

```vue
<script lang="ts">
import { getUserById } from '../api'
import { defineLoader } from '@vue-router'

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

# Motivation

There are currently too many ways of handling data fetching with vue-router and all of them have problems:

- With navigation guards:
  - using `meta`: complex to setup even for simple cases, too low level for such a common case
  - using `onBeforeRouteUpdate()`: missing data when entering tha page
  - `beforeRouteEnter()`: non typed and non-ergonomic API with `next()`, requires a data store (pinia, vuex, apollo, etc)
  - using a watcher on `route.params...`: component renders without the data (doesn't work with SSR)

The goal of this proposal is to provide a simple yet configurable way of defining data loading in your application that is easy to understand and use. It should also be compatible with SSR and allow extensibility.

# Detailed design

This design is possible though [unplugin-vue-router](https://github.com/posva/unplugin-vue-router):

- Check named exports in page components to set a meta property in the generated routes
- Adds a navigation guard that resolve loaders
- Implement a `defineLoader()` composable

`defineLoader()` takes a function that returns a promise and returns a composable that **can be used in any component**, not only in the one that defines it. We call these _loaders_. Loaders **must be exported by page components** in order for them to get picked up and executed during the navigation. They receive the target `route` as an argument and must return a Promise of an object of properties that will then be directly accessible **as refs** when accessing

## Parallel Fetching

By default, loaders are executed as soon as possible, in parallel. This scenario works well for most use cases where data fetching only requires params or nothing at all.

## Sequential fetching

Sometimes, requests depend on other fetched data (e.g. fetching additional user information). For these scenarios, we can simply import the other loaders and use them **within a different loader**:

Call the loader inside the one that needs it, it will only be fetched once

```ts
export const useUserFriends = defineLoader(async (route) => {
  const { user, isLoaded() } = useUserData()
  await isLoaded()
  const friends = await getFriends(user.value.id)
  return { user, friends }
})
```

Note that two loaders cannot use each other as that would create a _dead lock_.

Alternatives:

- Allowing `await getUserById()` could make people think they should also await inside `<script setup>` and that would be a problem because it would force them to use `<Suspense>` when they don't need to.

## Combining loaders

```js
export const useLoader = combineLoaders(useUserLoader, usePostLoader)
```

## Loader reusing

Each loader has its own cache attached to the application

```ts
// the map key is the function passed to defineLoader
app.provide('loaderMap', new WeakMap<Function>())

// then the dataLoader

export const useLoader = (loader: Function) => {
  const loaderMap = inject('loaderMap')
  if (!loaderMap.has(loader)) {
    // create it and set it
  }
  // reuse the loader instance
}
```

We could transform the `defineLoader()` calls to include an extra argument (and remove the first one if it's a route path) to be used as the `key` for the loader cache.
NOTE: or we could just use the function name + id in dev, and just id number in prod

## Usage outside of page components

Loaders can be **only exported from pages**. That's where the plugin picks it up, but the page doesn't even need to use it. It can be used in any component by importing the function, even outside of the scope of the page components.

## TypeScript

Types are automatically generated for the routes by [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) and can be referenced with the name of each route to hint `defineLoader()` the possible values of the current types:

```vue
<script lang="ts">
import { getUserById } from '../api'
import { defineLoader } from '@vue-router'

export const useUserData = defineLoader('/users/[id]', async (route) => {
  //                                ^ autocompleted
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

Testing the pages is now easy

## Named views

When using vue router named views, each named view can have their own loader but note any navigation to the route will trigger **all loaders from all page components**.

Note: a named view can be declared by appending `@name` at the end of the file name:

```
src/pages/
└── users/
    ├── index.vue
    └── index@aux.vue
```

This creates a `components: { default: ..., aux: ... }` entry in the route config.

## Reusing / Sharing loaders

Loaders can be defined anywhere and imported and used (only in page components) when necessary. This allows to define loaders anywhere or even reuse loaders from a different page.
Any page component with named exports will be marked with a symbol to pick up any possible loader in a navigation guard.

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
  { lazy: true }
)
</script>

<script setup>
// in this scenario, we no longer have a `user` property since `useUserData()` returns synchronously before the loader is resolved
const { data, pending, error } = useUserData()
//      ^ Ref<{ user: User } | undefined>
</script>
```

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

Note that lazy loaders can only control their own blocking mechanism. They can't control the blocking of other loaders. If multiple loaders are being used and one of them is blocking, the navigation will be blocked until all of them are resolved.

Or a function to decide upon navigation / refresh:

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

## Refreshing the data

### With navigation

### Manually

Manually call

## Error handling

If a request fails, the user can catch the error in the loader and return it differently

## SSR

### Avoiding double fetch on the client

## Performance

When fetching large data sets, it's convenient to mark the fetched data as _raw_ before returning it:

```ts
export const useBookCatalog = defineLoader(async () => {
  const books = markRaw(await getBookCatalog())
  return { books }
})
```

[More in Vue docs](https://vuejs.org/api/reactivity-advanced.html#markraw)

An alternative would be to use `shallowRef()` instead of `ref()` but that would prevent users from modifying the returned value and overall less convenient. Having to use `markRaw()` seems like a good trade off in terms of API and performance.

## HMR

When changing the `<script>` the old cache is transferred and refreshed.

## Extending `defineLoader()`

It's possible to extend the `defineLoader()` function to add new features such as a more complex cache system. TODO:

## Limitations

- Injections (`inject`/`provide`) cannot be used within a loader
- Watchers and other composables can be used but **will be created in a detached effectScope** and therefore never be released. **This is why, using composables should be avoided**.

# Drawbacks

This solution is not a silver bullet but I don't think one exists because of the different data fetching strategies and how they can define the architecture of the application.

- Less intuitive than just awaiting something inside `setup()`
- Can only be used in page components
- Requires an extra `<script>` tag but only for views

# Alternatives

- Adding a new `<script loader>` similar to `<script setup>`:

  ```vue
  <script lang="ts" loader="useUserData">
  import { getUserById } from '@/api/users'
  import { useRoute } from '@vue-router' // could be automatically imported

  const route = useRoute()
  // any variable created here is available in useLoader()
  const user = await getUserById(route.params.id)
  </>

  <script lang="ts" setup>
  import { useLoader } from '@vue-router'

  const { user, pending, error } = useUserData()
  </>
  ```

  Is exposing every variable a good idea?

- Using Suspense to natively handle `await` within `setup()`. [See other RFC](#TODO).

## Naming

Variables could be named differently and proposals are welcome:

- `pending` (same as Nuxt) -> `isPending`, `isLoading`
- `export const loader` (not a verb because it's not meant to be called like a function) -> `export const load`

# Adoption strategy

Introduce this as part of [unplugin-vue-router](https://github.com/posva/unplugin-vue-router) to test it first and make it part of the router later on.

# Unresolved questions

- Should there be a way to handle server only loaders?
- Is `useNuxtApp()` usable within loaders?
- Is there anything needed besides the `route` inside loaders?
