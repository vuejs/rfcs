- Start Date: 2019-05-10
- Target Major Version: Vue (2.x / 3.x) Vue Router (3.x / 4.x)
- Reference Issues: https://github.com/vuejs/vue-router/issues/2040
- Implementation PR: (leave this empty)

# Summary

Customize current _exact_ matching behaviour by allowing a string value to the `exact` prop.

# Basic example

```vue
<!-- same as `exact` boolean prop -->
<router-link to="/some">/some</router-link>
<!-- matches only the path, eg: `/?query`, `#foo`, `?q=hey`. But not `/a`, `/some` -->
<router-link to="/" exact="path">Home</router-link>
<!-- matches the path and query eg: `?q=hey&a=b`, `?a=b&q=hey`, But not `/` or `?q=hey` -->
<router-link to="/?q=hey&a=b" exact="path+query">Home</router-link>
```

# Motivation

Current `exact` behavior is not flexible enough. A boolean value can only handle two cases but there are many more when dealing with query params or hashes as part of the link and making them responsible for marking the link as active.

# Detailed design

Right now, the active state can either be _exact_ or normal by passing the `exact` prop. However this is quite limiting for path containing query params or a hash which are taked into account when exact matching.
If the `exact` prop is not supplied, the link is considered active when it is included by the current location path as a string:
eg: `<router-link to="/">Home</router-link>`/`<router-link :to="{ name: 'home' }">Home</router-link>` would be highlighted by any link (parent-child relationship is not necessary). The same goes for these two routes:

```js
const routes = [{ path: '/some' }, { path: '/some/option' }]
```

Both links are actives while being on `/some/option`:

```vue
<router-link to="/some">/some</router-link>
<router-link to="/some/option">/some/option</router-link>
```

If `/some/option` is matched, the link `/some` will be marked as active but if another link like `/some-option` was matched (note the `/` replaced by `-`), which is not an _append_ of `/some`, `/some` would not be marked as active.

If `exact` is supplied, the link matches only if it matches the `record` matched by current location and link component **as well as** query + hash params.
To allow an `exact` match to also allow different query params and hash params, we could allow a string value for the `exact` match so we don't break existing behavior:

Imagine being in `/path?foo=1#bar`, here is the matching table based on the value of `exact`. `'all'` would be the equilavent of `true`

| Path \ `exact` value  | `all` | `path` | `query` | `hash` | `path+query` |
| --------------------- | ----- | ------ | ------- | ------ | ------------ |
| /path                 | ❌    | ✅     | ❌      | ❌     | ❌           |
| /path/child           | ❌    | ❌     | ❌      | ❌     | ❌           |
| /path?foo=1#bar       | ✅    | ✅     | ✅      | ✅     | ✅           |
| /path?foo=1           | ❌    | ✅     | ✅      | ❌     | ✅           |
| /path#bar             | ❌    | ✅     | ❌      | ✅     | ❌           |
| /path?foo=1#foo       | ❌    | ✅     | ✅      | ❌     | ✅           |
| /path?foo=2#bar       | ❌    | ✅     | ❌      | ❌     | ❌           |
| /path?foo=1&bar=3#bar | ❌    | ✅     | ❌      | ❌     | ❌           |

When using the `+` as in `path+query` you are expecting both, the path and the query section to exactly match. Using `query+hash` would match both the query and hash but not care about the `path` (is this really useful?)

# Drawbacks

- Learning cost

# Alternatives

- Adding an `exact-path` prop. This could be confusing: should it be added alongside `exact`? It doesn't allow having an exact match only on the hash or the query
  What other designs have been considered? What is the impact of not doing this?

# Adoption strategy

- Provide a backwards compatible version with deprecation messages and break changes only in v4

# Unresolved questions

- What about partial queries? `{ bar: '2' }` is included in `{ foo: 'a', bar: '2' }`, should it also match the url? should it be handled in userland? Should be provide helpers? Should we make the default behaviour like the proposal for `exact` string prop but allow including instead of exact same values? I think we should keep exact matches only and let the user do any further custom matching using the scoped-slot proposal
