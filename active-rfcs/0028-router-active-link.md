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
// this is only necessary if we care about matching the query
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
It also makes more sense from a navigation perspective for an active link not to trigger a new navigation (except in the case of nested routes where it could be a navigation).

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
      { path: 'child/:id2' },
      { path: 'child-second/:id2' }
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

## Unrelated but similiar routes

Routes that are unrelated from a record point of view but share a common path are no longer active.

E.g., given these routes:

```js
const routes = [
  { path: '/movies' },
  { path: '/movies/new' },
  { path: '/movies/search' }
]
```

If the current route is `/movies/new`, these links will be active:

| url            | active | exact active |
| -------------- | ------ | ------------ |
| /movies        | ❌     | ❌           |
| /movies/new    | ✅     | ✅           |
| /movies/search | ❌     | ❌           |

Note: **This behavior is different from actual behavior**.

It's worth noting, it is possible to nest them to still benefit from links being _active_:

```js
// Vue 3
import { h } from 'vue'
import { RouterView } from 'vue-router'

const routes = [
  {
    path: '/movies',
    // we need this to render the children (see note below)
    component: { render: () => h(RouterView) },
    // for vue 2 use render: h => h('RouterView')
    children: [
      { path: 'new' },
      // different child
      { path: 'search' }
    ]
  }
]
```

If the current route is `/movies/new`, these links will be active:

| url            | active | exact active |
| -------------- | ------ | ------------ |
| /movies        | ✅     | ❌           |
| /movies/new    | ✅     | ✅           |
| /movies/search | ❌     | ❌           |

Note: To make this easier to use, we could maybe allow `component` to be absent and internally behave as if there where a `component` option that renders a `RouterView` component

## Alias

Given that an alias is only a different `path` while keeping everything else on a record, it makes sense for aliases to be active when the `path` they are aliasing is matched and vice versa.

E.g., given these routes:

```js
const routes = [{ path: '/movies', alias: ['/films'] }]
```

If the current route is `/movies` **or** `/films`, both links will be active:

| url     | active | exact active |
| ------- | ------ | ------------ |
| /movies | ✅     | ✅           |
| /films  | ✅     | ✅           |

### Nested aliases

The behavior is similar when dealing with nested children of an aliased route.

E.g., given these routes:

```js
const routes = [
  {
    path: '/parent/:id',
    alias: '/p/:id',
    children: [
      // empty child
      { path: '' },
      // child with id
      { path: 'child/:id2', alias: 'c/:id2' }
    ]
  }
]
```

If the current route is `/parent/1/child/2`, `/p/1/child/2`, `/p/1/c/2`, or, `/parent/1/c/2` these links will be active:

| url               | active | exact active |
| ----------------- | ------ | ------------ |
| /parent/1/child/2 | ✅     | ✅           |
| /parent/1/c/2     | ✅     | ✅           |
| /p/1/child/2      | ✅     | ✅           |
| /p/1/c/2          | ✅     | ✅           |
| /p/1/child/3      | ❌     | ❌           |
| /parent/1/child/3 | ❌     | ❌           |
| /parent/1         | ✅     | ❌           |
| /p/1              | ✅     | ❌           |
| /parent/2         | ❌     | ❌           |
| /p/2              | ❌     | ❌           |

### Absolute nested aliases

Nested children can have an absolute `path` by making it start with `/`, in this scenario the same rules apply. Given these routes:

E.g., given these routes:

```js
const routes = [
  {
    path: '/parent/:id',
    alias: '/p/:id',
    name: 'parent',
    children: [
      // empty child
      { path: '', alias: ['alias', '/p_:id'], name: 'child' },
      // child with absolute path. We need to add an `id` because the parent needs it
      { path: '/p_:id/absolute-a', alias: 'as-absolute-a' },
      // same as above but the alias is absolute
      { path: 'as-absolute-b', alias: '/p_:id/absolute-b' }
    ]
  }
]
```

If the current route is `/p_1/absolute-a`, `/p/1/as-absolute-a`, or, `/parent/1/as-absolute-a`, these links will be active:

| url                  | active | exact active |
| -------------------- | ------ | ------------ |
| /p/1/as-absolute-a   | ✅     | ✅           |
| /p_1/absolute-a      | ✅     | ✅           |
| /parent/1/absolute-a | ✅     | ✅           |
| /parent/2/absolute-a | ❌     | ❌           |
| /parent/1/absolute-b | ❌     | ❌           |
| /p/1/absolute-b      | ❌     | ❌           |
| /p_1/absolute-b      | ❌     | ❌           |
| /parent/1            | ✅     | ❌           |
| /p/1                 | ✅     | ❌           |
| /parent/1/alias      | ✅     | ❌           |
| /p/1/alias           | ✅     | ❌           |
| /p_1                 | ✅     | ❌           |
| /parent/2            | ❌     | ❌           |
| /p/2                 | ❌     | ❌           |

Notice how the empty `path` record is _active_ but not _exact active_ differently from the other child `/p/1/absolute-b`. All its aliases are _active_ as well because they are aliases of an empty `path`. If it was the other way around: the `path` wasn't empty but one of the aliases was an empty `path`, then **none** of them would be active because the original `path` takes precedence over aliases.

### Named nested routes

If the url is resolved through the `name` of the parent, in this case `parent`, **it will not include the empty path child route**. This is important because they both resolve to the same url but when used in `router-link`'s `to` prop, they would yield different results when it comes to being _active_ and/or _exact active_. This is consistent with what they render being different and the rest of the active behavior.

E.g., given the routes from the previous example, **if the current location is `/parent/1` and, both the parent and child views are rendering, meaning we are effectively at `{ name: 'child' }` and not at `{ name: 'parent' }`**, here is a similar table to the ones before but also including `to`:

| `to`'s value                              | resolved url              | active | exact active |
| ----------------------------------------- | ------------------------- | ------ | ------------ |
| `{ name: 'parent', params: { id: '1' } }` | `/parent/1` (parent)      | ✅     | ❌           |
| `'/parent/1'`                             | `/parent/1` (child)       | ✅     | ✅           |
| `{ name: 'child', params: { id: '1' } }`  | `/parent/1` (child)       | ✅     | ✅           |
| `'/p_1'`                                  | `/p_1` (child)            | ✅     | ✅           |
| `'/parent/1/alias'`                       | `/parent/1/alias` (child) | ✅     | ✅           |

But **if the current location is `{ name: 'parent' }`**, it will still yield the same url, `/parent/1`, but a different table:

| `to`'s value                              | resolved url              | active | exact active |
| ----------------------------------------- | ------------------------- | ------ | ------------ |
| `{ name: 'parent', params: { id: '1' } }` | `/parent/1` (parent)      | ✅     | ✅           |
| `'/parent/1'`                             | `/parent/1` (child)       | ❌     | ❌           |
| `{ name: 'child', params: { id: '1' } }`  | `/parent/1` (child)       | ❌     | ❌           |
| `'/p_1'`                                  | `/p_1` (child)            | ❌     | ❌           |
| `'/parent/1/alias'`                       | `/parent/1/alias` (child) | ❌     | ❌           |

## Repeated params

With repeating params like

- `/articles/:id+`
- `/articles/:id*`

**All params** must match, with the same exact order for a link **to be both**, _active_ and _exact active_.

## `exact` prop

Before these changes, `exact` worked by matching the whole location. Its main purpose was to get around the `/` caveat but it also checked `query` and `hash`.
Because of this, with the new active behavior, the [`exact` prop](https://router.vuejs.org/api/#exact) only purpose would be for a link to display `router-link-active` only if `router-link-exact-active` is also present. But this isn't really useful anymore as we can directly target the element using the `router-link-exact-active` class.
Because of this, I think the `exact` prop can be removed from `router-link`. This outdates https://github.com/vuejs/rfcs/pull/37 while still introducing the behavior of `exact` for the path section of the loction like explained above.

Some users will probably have to change the class used in CSS from `router-link-exact-active` to `router-link-active` to adapt to this change.

# Drawbacks

- This is not backwards compatible. It's probably worth adding `exact-path` in Vue Router v3.
- Users using the `exact` prop will have to rely on the `router-link-exact-active` or use the `exact-active-class` prop.
- The function `includesQuery` must be added by the user.

# Alternatives

- Leaving the `exact` prop to only apply `router-link-active` when `router-link-exact-active` as also applied.
- Keep _active_ behavior of doing an inclusive match of `query` and `hash` instead of just relying on the params.
- Changing _active_ behavior to only apply to the `path` section of a route (this includes params) and ignore `query` and `hash`.

# Adoption strategy

- Add `exact-path` to mitigate existing problems in v3
