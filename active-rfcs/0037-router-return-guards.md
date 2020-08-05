- Start Date: 2020-06-08
- Target Major Version: Vue Router 3 and 4
- Reference Issues: https://github.com/vuejs/rfcs/issues/177
- Implementation PR: https://github.com/vuejs/vue-router-next/pull/343

# Summary

Allow navigation guards to return the value **or a Promise** of it instead of calling `next`:

```js
// change
router.beforeEach((to, from, next) => {
  if (!isAuthenticated) next(false)
  else next()
})

// into
router.beforeEach(() => isAuthenticated)
```

Since we can check the amount or arguments the navigation guard has, we can automatically know if we should look at the returned value or not. **This is not a breaking change**.

# Motivation

- Avoid forgetting to call `next`
- Avoid the problem of calling `next` multiple times (since we can only return once)
- `return` is more idiomatic since it can only be called once
- Avoid the need of 3 arguments in the function if they are not all used

# Detailed design

If 2 arguments are passed to a navigation guard, Vue Router will look at the returned value as if it was passed to `next`.

## Validate/cancel a navigation

To validate a navigation, you currently have to call `next()` or `next(true)`. Instead, you can not return anything (same as explicitly returning `undefined`) or return `true`. To cancel the navigation you could call `next(false)`, now you can explicitly `return false`:

```js
router.beforeEach((to) => {
  if (to.meta.requiresAuth && !isAuthenticated) return false
})

// with async / await
router.beforeEach(async (to) => {
  return await canAccessPage(to)
})
```

## Redirect to a different location

We can redirect to a location by returning the same kind of object passed to `router.push`:

```js
router.beforeEach((to) => {
  if (to.meta.requiresAuth && !isAuthenticated)
    return {
      name: 'Login',
      query: {
        redirectTo: to.fullPath,
      },
    }
})

// with async / await
router.beforeEach(async (to) => {
  if (!(await canAccessPage(to))) {
    return {
      name: 'Login',
      query: {
        redirectTo: to.fullPath,
      },
    }
  }
})
```

## Errors

Unexpected errors can still be thrown synchronously or synchronously:

```js
router.beforeEach((to) => {
  throw new Error()
})

// with async / await
router.beforeEach(async (to) => {
  throw new Error()
})

// with promises
router.beforeEach((to) => {
  return Promise.reject(new Error())
})
```

# Drawbacks

- Vue Router 3 might preset a higher implementation cost

# Alternatives

- This could be implemented with a function helper but the idea is to shift the way we write navigation guards.

What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

- Add both syntaxes to Vue Router 4 (https://github.com/vuejs/vue-router-next/pull/343/files)
- A codemod should be able to handle the conversion even though it's not a breaking change
