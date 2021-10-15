- Start Date: 2021-10-15
- Target Major Version: Router 4.x
- Reference Issues: https://github.com/vuejs/vue-router/issues/2243
- Implementation PR: (leave this empty)

# Summary

- Allow the user to pass a `state` property alongside `path`, `query`, and other properties to persist to `history.state`.
- Allow the user to read the `history.state` directly at `this.$route.state`.

# Basic example

Programmatic navigation:

```js
router.push({ name: 'Details', state: { showModal: true } })
router.replace({ state: { showModal: true } })
```

Declarative:

```vue
<router-link
  :to="{ name: 'Details', state: { showModal: true } }"
>Show Details</router-link>
```

# Motivation

Passing _state_ through the `history` API is a native feature that is currently hard to use when using Vue Router. While it has its limitations, it has many useful usecases like showing modals and can be use as a source of truth for state that is specific to certain locations and **should be persisted** across navigations when **coming back** to a previously visited page.

Currently, this can be achieved most of the times with

```js
// check for the navigation to succeed
if (!(await router.push('/somewhere'))) {
  history.replaceState({ ...history.state, ...newState }, '')
}
```

It currently cannot be achieved if the current location is the same and the only thing we want to do is _modify_ the state.

The router should facilitate using the features of the History API but currently it turns out to make the task of _writing to `history.state`_ difficult or impossible (e.g. same location navigation)

# Detailed design

## Writing to `history.state`

Vue Router 4 already uses `history.state` internally to detect navigation direction and revert UI initiated navigations such as the back and forward button. In order to not interfere with the information stored by it, it should save the state passed by the user to a nested property:

```js
// somewhere inside the router code
history.pushState({ ...routerState, userState: state }, '', url)
```

### Duplicated navigations

By default, the router avoids any duplicated navigation (e.g. clicking multiple times on the same link) or calling `router.push('/somewhere')` when we are already at `/somewhere`. When `state` is passed to `router.push()` (or `router.replace()`), the router should **always create a new navigation**. This creates a hidden way to force a navigation to the same location and also the possibility to have multiple entries on the history stack that point to the same URL but this should be fine as they should contain different state.

1. User goes to `/search`
2. User clicks on button that does `router.push({ state: { searchResults: [] }})`
3. User stays at `/search` but the page can use the passed state to display a different version
4. User clicks the _back_ button, they stay at `/search` but see a different version of the page

### Invalid state properties

Since [the state must be serializable](https://developer.mozilla.org/en-US/docs/Web/API/History/pushState), some _key_ or _property_ values are invalid and should be avoided (e.g. DOM nodes or complex objects, functions, Symbols). Vue Router **won't touch the state given by the user** and pass it _as is_ to `history.pushState()`. The developer is responsible for this and must be aware that browsers might treat some Data Structures differently.

### SSR

Since this feature only works with the History API, any given `state` property passed to `router.push()` will be ignored during SSR by the _Memory History_ implementation.

## Reading the state

It would be convenient to be able to read the `history.state` directly from the current route because that would make it _reactive_ and allow watching or creating computed properties based on it:

```js
const route = useRoute()

const showModal = computed(() => route.state.showModal)
```

```vue
<Modal v-if="$route.state.showModal" />
```

For convenience reasons, the `route.state` property should be an empty object by default.

This introduces **a new TS interface to represent the current location** as the History API only allows reading from the current entry. Therefore **`from.state` is unavailable** in Navigation guards while `to.state` can be available:

```ts
router.beforeEach((to, from) => {
  to.state // undefined | unknown
  from.state // TS Error property doesn't exist
})
```

# Drawbacks

- The History API has its own limitations and inconsistencies among browsers and they sometimes vary (e.g. the way state is persisted to disk and how objects are cloned). This could be a foot shot if not documented properly in terms of usage. For instance, it should be avoided to store big amounts of data that should go in component state or in a store
- Making `route.state` retrieve only `history.state.userState` allows us to not expose the information stored by the router (since it's not public API) but also doesn't allow information stored in the `history.state` by other libraries. I think this is okay because the user can create a computed property to read from those properties with `computed(() => route && history.state.myOwnProperty)`.

# Alternatives

- The `route.state` property could be `undefined` when not set.
- Letting `route.state` be the whole `history.state` instead of what the user passed

# Adoption strategy

Currently Vue Router 4 allows passing a `state` property to `router.push()` but the API is not documented and marked as `@internal` and should therefore not be used. Other than that, this API is an addition.

# Unresolved questions
