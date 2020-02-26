- Start Date: 2020-01-27
- Target Major Version: Vue 3 Router 4
- Reference Issues: [issues](https://github.com/vuejs/vue-router/issues?q=is%3Aopen+is%3Aissue+label%3A%22group%5Bdynamic+routing%5D%22)
- Implementation PR:

# Summary

Introducing an API that allows adding and removing route records while the router is working

- `router.addRoute(route: RouteRecord)` Add a new route
- `router.removeRoute(name: string | Symbol)` Remove an existing route
- `router.hasRoute(name: string | Symbol): boolean` Check if a route exists
- `router.getRoutes(): RouteRecord[]` Get the current list of routes

# Basic example

Given a router instance in a running app:

```ts
router.addRoute({
  path: '/new-route',
  name: 'NewRoute',
  component: NewRoute
})

// add to the children of an existing route
router.addRoute('ParentRoute', {
  path: 'new-route',
  name: 'NewRoute',
  component: NewRoute
})

router.removeRoute('NewRoute')

// normalized version of the records added
const routeRecords = router.getRoutes()
```

# Motivation

Dynamic routing is a feature that enables applications to build its own routing system. A usecase of this is the vue-cli ui, that allows adding graphical plugins which can have their own interfaces.

The current version of Vue Router only supports adding a new absolute route. The idea of this RFC is to add the missing functions to allow dynamic routing. Thinking about the different ways this API can be shaped and what is best for the future of Vue Router. Currently, it's impossible to achieve the example described above without hacks or creating a whole new Router instance.

In theory this could also improve Developer experience by changing routes in place with Hot Module Replacement.

Note: Dynamic routing only makes sense after implementing path ranking (automatic route order that allows route records to be added in any order but still be matched correctly). Most of the cost of adding this feature is on the path scoring side.

# Detailed design

## `addRoute`

- Allow adding an absolute route
- Allow adding a child route
- How should conflicts be handled?
- What should be returned by the function?

The code samples are written in TS to provide more information about the shape of the object expected:

`RouteRecord` is the same kind of object used in the `routes` option when instantiating the Router. To give you an idea, it looks like (https://router.vuejs.org/api/#routes). Important to note, this object is consistent with the `routes` options, since the `RouteRecord` type might have minor changes in the future version of Vue Router 4.

```ts
const routeRecord: RouteRecord = {
  path: '/new-route',
  name: 'NewRoute',
  component: NewRoute
}
```

There are different options for what to return from `addRoute`.

Returning a function that allows removing the route is useful for unamed routes.

```ts
const removeRoute = router.addRoute(routeRecord)

removeRoute() // removes the route record
// or
router.removeRoute('NewRoute') // because names are unique
```

If we are on `/new-route`, adding the record will **not** trigger a _replace_ navigation. The user is responsible for forcing a navigation by calling `router.push` or `router.replace` with the current location `router.replace(router.currentRoute.value.fullPath)`. Using the string version of the location ensures that a new location is resolved instead of the old one. If the route is added inside of a navigation guard, make sure to use the value from the `to` parameter:

```js
router.beforeEach((to, from, next) => {
  // ...
  router.addRoute(newRoute)
  next(to.fullPath)
  // ...
})
```

Because the Dynamic Routing API is an advanced API that most users won't directly use, having this split allows more flexible and optimized behavior. The recommended approach to add routes will still be the configuration based one.

_For Alternatives, please check [alternatives](#alternatives)_

### Conflicts

When adding a route that has the same name as an existing route, it should replace the existing route. This is the most convenient version, because it allows replacing new routes without having to explicitely remove the old ones **when they are using the same name**.

Alternatives:

- Fail and throw an error that asks the user to replace it first: less convenient
- Not warn the user but still replace it: could be prone to difficult to debug errors

### Nested routes

Depending on what is returned by `addRoute`, a nested route can be added by referencing the name of an existing route. This forces parent routes to be named.

```ts
router.addRoute('ParentRoute', routeRecord)
```

### Signature

```ts
interface addRoute {
  (parentName: string | Symbol, route: RouteRecord): () => void
  (route: RouteRecord): () => void
}
```

## `removeRoute`

Removing a route removes all its children as well. As with `addRoute`, it's necessary to trigger a new navigation in order to update the current displayed view by _RouterView_.

```ts
interface removeRoute {
  (name: string | Symbol): () => void
}
```

## `hasRoute`

Checks if a route exists:

```ts
interface hasRoute {
  (name: string | Symbol): boolean
}
```

## `getRoutes`

Allows reading the list of normalized route records:

```ts
interface getRoutes {
  (): RouteRecordNormalized[]
}
```

What is present in RouteRecordNormalized is yet to be decided, but contains at least all existing properties from a RouteRecord, some of them normalized (like `components` instead of `component` and an `undefined` name).

# Drawbacks

- This API increases vue-router size. To make it treeshakable will require allowing the matcher (responsible for parsing `path` and doing the path ranking) to be extended with simpler versions, which are quite complex to write but we could instead export different versions of the matcher and allow the user specifying the matcher in a leaner version of the router.

# Alternatives

## `addRoutes`

- A promise that resolves or rejects based on the possible navigation that adding the record might trigger (e.g. being on `/new-route` before adding the route)

```ts
window.location.pathname // "/new-route"
// 1. The promise of a function that allows removing the route record
const removeRoute = await router.addRoute(routeRecord)
// 2. The same kind of promise returned by `router.push`.
const route = await router.addRoute(routeRecord)

removeRoute() // removes the route record
// or
router.removeRoute('NewRoute') // names are unique
```

- A reference to the record added (a normalized copy of the original record)

```ts
// js object (same reference hold by the router)
const normalizedRouteRecord = router.addRoute(routeRecord)
router.removeRoute(normalizedRouteRecord)
```

## `getRoutes`

There could also be a reactive property with the routes, but this would allow using them in templates, which in most scenarios is an application level feature which could and should handle things the other way around, **having a reactive source of route records that are syncronized with the router**. Doing this at the application level avoids adding the cost for every user.

# Adoption strategy

- Deprecate `addRoutes` and add `addRoute` to Vue Router 3. Other methods are new.
