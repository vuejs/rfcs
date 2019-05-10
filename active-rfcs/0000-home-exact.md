- Start Date: 2019-05-10
- Target Major Version: (2.x / 3.x) (Breaking)
- Reference Issues: https://github.com/vuejs/vue-router/issues/454
- Implementation PR:

# Summary

Change default behavior of `router-link-active` class for the `'/'` path, so it's only applied on the `<router-link>` when the current location is exactly `"/"`.

# Basic example

Before:

```html
<router-link to="/" exact>Home</router-link>
<router-link to="/about">About</router-link>
```

After:

```html
<router-link to="/">Home</router-link>
<router-link to="/about">About</router-link>
```

# Motivation

The big benefits of this change is to make the `.router-link-active` class just work out of the box for every router-link.

Downsides of current implementation:

- Learning how to use the `.router-link-active` class is unnecessary made more difficult with this flaw in the class behavior.
- The dynamic `.router-link-active` class is effectively static on the root path so useless, unless `exact` prop is used.
- It doesn't make sense almost all of the time to have all routes being child of the `'/'` route.

# Detailed design

If the resolved route path is `'/'`, then `exact` default value is `true`.

# Drawbacks

This is a breaking change.

Migration:
- Remove `exact` prop on router links targeting the root path.

# Alternatives

# Adoption strategy

# Unresolved questions

