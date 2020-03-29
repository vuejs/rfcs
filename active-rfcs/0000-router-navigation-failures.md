- Start Date: 2020-03-26
- Target Major Version: Vue Router v4
- Reference Issues: https://github.com/vuejs/vue-router/issues/2833, https://github.com/vuejs/vue-router/pull/3047, https://github.com/vuejs/vue-router/issues/2932, https://github.com/vuejs/vue-router/issues/2881
- Implementation PR:

# Summary

- Explicitely define what a navigation failure is and how and where we can catch them.
- Change when the Promised-based `router.push`(and `router.replace` by extension) resolves and rejects.
- Make `router.push` consistent with `router.afterEach` and `router.onError`.

# Basic example

## `router.push`

If there is an unhandled error or an error is passed to `next`:

```js
// any other navigation guard
router.beforeEach((to, from, next) => {
  next(new Error())
  // or
  throw new Error()
  // or
  return Promise.reject(new Error())
})
```

Then the promise returned by `router.push` rejects:

```js
router.push('/url').catch(err => {
  // ...
})
```

In **all** other cases, the promise is resolved. We can know if the navigation failed or not by checking the resolved value:

```js
router.push('/dashboard').then(failure => {
  if (failure) {
    failure instanceof Error // true
    failure.type // NavigationFailure.canceled
  }
})
```

## `router.afterEach`

It's the global equivalent of `router.push().then()`

## `router.onError`

It's the global equivalent of `router.push().catch()`

# Motivation

The current behavior of Vue Router regarding the promise returned by `push` is inconsistent with `router.afterEach` and `router.onError`. Ideally, we should be able to catch all succeeded and failed navigations globally and locally but we can only do it locally.

- `onError` is only triggered on thrown errors and `next(new Error())`
- `afterEach` is only called if there is a navigation
- `redirect` should behave the same as `next('/url')` in a Navigation guard

The differences between the Promise resolution/rejection vs `router.afterEach` and `router.onError` are inconsistent and confusing.

# Detailed design

One of the main points is to be able to consistently handle failed navigations globally and locally:

- Failed Navigation:
  - Triggers `router.afterEach`
  - Resolves the `Promise` returned by `router.push`
- Uncaught Errors, `next(new Error())`
  - Triggers `router.onError`
  - Rejects the `Promise` returned by `router.push`

It's important to note there is no overlap in these two groups: if there is an uhandled error in a Navigation Guard, it will trigger `router.onError` as well as rejecting the `Promise` returned by `router.push` but **will not trigger** `router.afterEach`. Cancelling a navigation with `next(false)` will not trigger `router.onError` but will trigger `router.afterEach`

## Changes to the Promise resolution and rejection

Navigation methods like `push` return a Promise, this promise **resolves** once the navigation succeeds **or** fail. It **rejects** only if there was an unhandled error. If it rejects, it will also trigger `router.onError`.

To differentiate a Navigation that succeeded from one that failed, the `Promise` returned by `push` will resolve to either `undefined` or a `NavigationFailure`:

```js
import { NavigationFailureType } from 'vue-router'

router.push('/').then(failure => {
  if (failure) {
    // Having an Error instance allows us to have a Stacktrace and trace back
    // where the navigation was cancelled. This will, in many cases, lead to a
    // Navigation Guard and the corresponding `next()` call that cancelled the
    // navigation
    failure instanceof Error // true
    if (failure.type === NavigationFailureType.canceled) {
      // ...
    }
  }
})

// using async/await
let failure = await router.push('/')
if (failure) {
  // ...
}
```

By not rejecting the `Promise` when the navigation fails, we are avoiding _Uncaught (in promise)_ errors while still keeping the possibility to check if the Navigation failed or not.

### Navigation Failures

There are a few different navigation failures, to be able to react differently in your code

Navigation failures can be differentiated through a `type` property. All possible values are hold by an `enum`, `NavigationFailureType`:

- `cancelled`: `next(false)` inside of a navigation guard or a newer navigation took place while another one was ongoing.
- `duplicated`: Navigating to the same location as the current one will cancel the navigation and not invoke any Navigation guard.

On top of the `type` property, navigation failures also expose `from` and `to` properties, exactly like `router.afterEach`

### Redirections

Redirecting inside of a navigation guard with `next('/url')` is not a navigation failure by itself, as the navigation still takes place and ends up somewhere. To detect a navigation, specially during SSR, there is a `redirectedFrom` property accessible on the `currentRoute`.

E.g.: imagine a navigation guard that redirects to `/login` when the user isn't authenticated:

```js
router.beforeEach((to, from, next) => {
  // redirect to the login page if the target location requires authentication
  if (to.meta.requiresAuth && !isAuthenticated) next('/login')
  else next()
})
```

When navigating to a location that requires authentication, we can retrieve the original location the user was trying to access via `redirectedFrom`:

```js
// user is not authenticated
await router.push('/profile/dashboard')

// `redirectedFrom` is a RouteLocationNormalized, like `currentRoute` but we are omitting
// most properties to make the example readable
router.currentRoute // { path: '/login', redirectedFrom: { path: '/profile/dashboard' } }
```

## Changes to `router.afterEach`

Since `router.afterEach` also triggers when a navigation fails, we need a way to know if the navigation succeeded or failed. To do that, we introduce an extra parameter that contains the same _failure_ we could find in a resolved navigation:

```js
import { NavigationFailureType } from 'vue-router'

router.afterEach((to, from, failure) => {
  if (failure) {
    if (failure.type === NavigationFailureType.canceled) {
      // ...
    }
  }
})
```

# Drawbacks

- Breaking change although migration is relatively simple and in many cases will allow the developer to remove existing code

# Alternatives

- Differentiating `next(false)` from navigations that get overriden by more recent navigations by defining another Navigation Failure

# Adoption strategy

- `afterEach` and `onError` are relatively simple to migrate, most of the time they are not used many times either.
- `router.push
