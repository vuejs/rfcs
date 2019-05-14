- Start Date: 2019-05-14
- Target Major Version: Vue (2.x / 3.x) Vue Router (3.x / 4.x)
- Reference Issues: https://github.com/vuejs/vue-router/issues/2040
- Implementation PR: (leave this empty)

# Summary

- Add `exact-path` that exactly matches **only** the path section (not taking into account the query and hash params)
- in v4 change `exact` to behave like `exact-path`
- in v4 deprecate `exact-path` in favor on `exact`

# Basic example

```vue
<!-- both are active if being on `/some?any=query`, but both are inactive if on `/some/nested/route` -->
<router-link to="/some?query=one" exact exact-path>One</router-link>
<router-link to="/some?query=two" exact exact-path>Two</router-link>
<!-- in v4 shows a deprecation message -->
<router-link to="/some?link" exact exact-path>Home</router-link>
<!-- in v4 behaves the same as `exact` + `exact-path` from v3 -->
<router-link to="/some?link" exact>Home</router-link>
```

# Motivation

Current `exact` behavior doesn't marry well with query params and hash. However, we believe that matching the fullPath of a link is less common than just matching the path without taking into account the query or the hash. This is why we want to change the current behaviour.

# Detailed design

Using `exact-path` changes the behavior of the `exact` prop, which means _both props are needed_. The idea behind this is for the prop to be completely useless in v4.

If one wants to do more complex match, they should use the scoped slot version (https://github.com/vuejs/rfcs/pull/34)

# Drawbacks

- Breaking change

# Alternatives

- Deprecating the `exact` prop instead and introduce a different prop for both v3 and v4

# Adoption strategy

- Add the prop in v3
- Document as breaking change
- Codemod to automatically remove the prop

# Unresolved questions
