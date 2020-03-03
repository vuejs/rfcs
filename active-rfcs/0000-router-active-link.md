- Start Date: 2020-02-28
- Target Major Version: Router v4
- Reference Issues: https://github.com/vuejs/vue-router/issues/2040
- Implementation PR:

# Summary

Change how `router-link-active` is applied to align more with the router concepts. Right now, a link is _active_ if the generated url is a partial version of the current location. This creates a few issues:

- Aliases are not (necessarily) matched
- While being on a child with a leading slash, a link to its parent won't show as active because the urls are different
- Query is used to make the link _active_

# Motivation

The issues mentioned above are hard or impossible to implement in userland whereas implementing the current behavior is very straightforward:

```vue
<router-link v-slot="{ route }">
  partial path: {{ $route.path.startsWith(route.path) }}
  <br />
  partial path + partial query: {{ $route.path.startsWith(route.path) && includesQuery($route.query, route.query) }}
</router-link>
```

```js
function includesQuery(outter, inner) {
  for (let key in inner) {
    let innerValue = inner[key]
    let outterValue = outter[key]
    if (typeof innerValue === 'string') {
      if (innerValue !== outterValue) return false
    } else {
      if (
        !Array.isArray(outterValue) ||
        outterValue.length !== innerValue.length ||
        innerValue.some((value, i) => value !== outterValue[i])
      )
        return false
    }
  }

  return true
}
```

They also make more sense from a router perspective because links will be active based on the route record being active or not. This is specially important for nested routes and aliases which currently are not active if they do not share a part of the current url with the router-link location.

# Detailed design

The active behavior should be more connected to the _Matcher_ part of the router, this means it should be related to what is being rendered by the current url, which is part of the _Route Record_. Therefore, the `query` and `hash` should not influence this. It also seems to be something [people are really interested in](https://github.com/vuejs/vue-router/issues/2040) but **I don't know if people actually benefit from active working with `query`**.

## `router-link-exact-active`

This change also affects the behavior of _exact active_. The class `router-link-exact-active` is applied only if the current route is **exactly** the same as the one from the link, from a _Matcher_ perspective. This means that `query` and `hash` are not relevant (same as _active_).

This new _active_ behavior completely eliminates the caveat of the `exact` prop for the _root link_ (`/`) and makes https://github.com/vuejs/rfcs/pull/36 unecessary without introducing an inconsistency.

## Nested Routes

It's worth noting that nested routes will match only if the params relevant to the rendered route are the same.

E.g., given these routes:

```js
const routes = [
  {
    path: '/parent/:id',
    children: [
      // empty child
      { path: '' },
      // child with id
      { path: 'child/:id' },
      { path: 'child-second/:id' }
    ]
  }
]
```

If the current route is `/parent/1/child/2`, these links will be active:

| url                      | active | exact active |
| ------------------------ | ------ | ------------ |
| /parent/1/child/2        | ✅     | ✅           |
| /parent/1/child/3        | ❌     | ❌           |
| /parent/1/child-second/2 | ❌     | ❌           |
| /parent/1                | ✅     | ❌           |
| /parent/2                | ❌     | ❌           |
| /parent/2/child/2        | ❌     | ❌           |
| /parent/2/child-second/2 | ❌     | ❌           |
|                          |        |              |

## Unrelated routes

Routes that are unrelated from a record point of view but share a common path are no longer active.

E.g., given these routes:

```js
const routes = [{ path: '/movies' }, { path: '/movies/new' }]
```

If the current route is `/parent/new`, these links will be active:

| url         | active | exact active |
| ----------- | ------ | ------------ |
| /movies     | ❌     | ❌           |
| /movies/new | ✅     | ✅           |

**This behavior is different from actual behavior**

## Repeated params

With repeating params like

- `/articles/:id+`
- `/articles/:id*`

**All params** must match, with the same exact order for a link to be both, _active_ and _exact active_.

## `exact` prop

Before these changes, `exact` worked by matching the whole location. Its main purpose was to get around the `/` caveat but it also checked `query` and `hash`.
Because of this, with the new active behavior, the [`exact` prop](https://router.vuejs.org/api/#exact) only purpose would be for a link to display `router-link-active` only if `router-link-exact-active` is also present. But this isn't really useful anymore as we can directly target the element using the `router-link-exact-active` class.
Because of this, I think the `exact` prop can be removed from `router-link`. This outdates https://github.com/vuejs/rfcs/pull/37 while still introducing the behavior of `exact` for the path section of the loction.

Some users will probably have to change the class used in CSS from `router-link-exact-active` to `router-link-active` to adapt to this change.

# Drawbacks

- This is not backwards compatible. It's probably worth adding `exact-path` in Vue Router v3
- Users using the `exact` prop will have to rely on the `router-link-exact-active` or use the `exact-active-class` prop.

# Alternatives

- Leaving the `exact` prop to only apply `router-link-active` when `router-link-exact-active`
- Keep _active_ behavior of doing an inclusive match of `query` and `hash`

# Adoption strategy

- Add `exact-path` to mitigate existing problems in v3
