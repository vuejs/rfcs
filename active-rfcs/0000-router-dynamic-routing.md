- Start Date: 2020-01-27
- Target Major Version: Vue 3 Router 4
- Reference Issues: [issues](https://github.com/vuejs/vue-router/issues?q=is%3Aopen+is%3Aissue+label%3A%22group%5Bdynamic+routing%5D%22)
- Implementation PR:

# Summary

Introducing an API that allows adding and removing route records while the router is working

- `router.addRoute(route: RouteRecord)`
- `router.removeRoute(name: string)`
- `router.getRoutes()`

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

Note: Dynamic routing only makes sense after implementing path ranking (automatic route order that allows route records to be add in any order but still be matched correctly). Most of the cost of adding this feature is on the path scoring side.

# Detailed design

## `addRoute`

- Allow adding an absolute route
- Allow adding a child route
- How should conflicts be handled?
- What should be returned by the function?

The code samples are written in TS to provide more information about what is the shape of the object expected:

`RouteRecord` is the same kind of object used in the `routes` option when instantiating the Router. To give you an idea, it looks like (https://router.vuejs.org/api/#routes) but what matters is that this object is consistent with the `routes` options since the `RouteRecord` type might have minor changes in the future version of Vue Router 4.

```ts
const routeRecord: RouteRecord = {
  path: '/new-route',
  name: 'NewRoute',
  component: NewRoute
}
```

There are different options of what to return from `addRoute`

Returning a function that allows removing the route is useful for unamed routes

```ts
const removeRoute = router.addRoute(routeRecord)

removeRoute() // removes the route record
// or
router.removeRoute('NewRoute') // because names are unique
```

If we are on `/new-route`, adding the record will trigger a _replace_ navigation and will trigger all navigation guards. In this scenario, if no catch-route (path: '\*') is present, `from` will be a route location with an empty `matched` array and missing all extra properties like `meta` and `name` while `to` will be the same actual location (same `path`, `fullPath`, `query` and `hash`) but with a non-empty array of `matched`.

_For Alternatives, please check [alternatives](#alternatives)_

### Conflicts

When adding a route that has the same name as an existing route, it should replace the existing route and warn about it (in dev only). This is the most convenient version because it allows replacing new routes without removing the old ones **when they are using the same name**.

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
  (parentName: string, route: RouteRecord): () => void
  (route: RouteRecord): () => void
}
```

## `removeRoute`

Removing a route removes all its children as well. If we were on the removed route, the router should re route, which could end up in the catch-all route (`path: '*'`).

```ts
interface removeRoute {
  (name: string): () => void
}
```

## `getRoutes`

Allows reading the list of normalized active route records:

```ts
interface getRoutes {
  (): RouteRecordNormalized[]
}
```

What is present in RouteRecordNormalized is yet to be precised but contains at least all existing properties from a RouteRecord, some of them normalized (like `components` instead of `component` and an `undefined` name)

# Drawbacks

- This API increases vue-router size. To make it treeshakable will require allowing the matcher (responsible for parsing `path` and doing the path ranking) to be extended with simpler versions, which are quite complex to write.
  Why should we _not_ do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people Vue
- integration of this feature with other existing and planned features
- cost of migrating existing Vue applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

## `addRoutes`

- A promise that resolves or reject based on the possible navigation that adding the record might trigger (e.g. being on `/new-route` before adding the route)

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

- We could also force the user to manually `push` after adding routes to avoid any promise based api on `addRoute`

- A reference to the record added (a normalized copy of the original record)

```ts
// js object (same reference hold by the router)
const normalizedRouteRecord = router.addRoute(routeRecord)
router.removeRoute(normalizedRouteRecord)
```

## `getRoutes`

There could also be a reactive property with the routes but this would allow using them in templates, which in most scenarios is an application level feature which could and should handle things the other way around **having a reactive source of route records that are syncronized with the router**. Doing this at the application level avoids adding the cost for every user

# Adoption strategy

This API is backwards compatible with what exists in Vue Router 3
