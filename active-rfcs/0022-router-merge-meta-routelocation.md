- Start Date: 2020-03-12
- Target Major Version: Router 4.x
- Reference Issues: N/A
- Implementation PR:

# Summary

When creating routes, you can attach arbitrary data with the `meta` property:

```js
{ path: '/profile', meta: { requiresAuth: true }}
```

It's then accessible in navigation guards and on `$route`:

```js
router.beforeEach((to, from, next) => {
  if (to.meta.requiresAuth && !auth.loggedIn()) next('/login')
  else next()
})
```

However, when dealing with nested routes, `meta` will contain only the matched route `meta`. You can still go through the array of `matched` records like [pointed out in the documentation](https://router.vuejs.org/guide/advanced/meta.html#route-meta-fields):

```js
router.beforeEach((to, from, next) => {
  if (to.matched.some(record => record.meta.requiresAuth))
  // ...
})
```

My proposal is to merge all matched routes meta, from parent to child so we can do `to.meta.requiresAuth`. I believe this is what Nuxt does but I couldn't find a link in the docs.

# Basic example

Given a nested route:

```js
{
  path: '/parent',
  meta: { requiresAuth: true, isChild: false },
  children: [
    { path: 'child', meta: { isChild: true }}
  ]
}
```

Navigating to `/parent/child` should generate a route with the following `meta` property:

```js
{ requiresAuth: true, isChild: true }
```

# Motivation

Most of the time, merging the `meta` property is what we need. I've never seen a case where I exclusively needed the most nested route `meta` property.

This would remove the need of doing `to.matched.some` to check if a `meta` field is present. Instead, it will be only necessary to use it to check overriden `meta` properties.

# Detailed design

The `meta` property is merged only at the first level, like `Object.assign` and the _spread operator_:

```js
{
  path: '/parent',
  meta: { nested: { a: true } },
  children: [
    { path: 'child', meta: { nested: { b: true }}}
  ]
}
```

yields the following `meta` object when at `/parent/child`:

```js
{
  nested: {
    b: true
  }
}
```

# Drawbacks

- This is technically a breaking change
