- Start Date: 2019-04-29
- Target Major Version: Vue (2.x / 3.x) Vue Router (3.x / 4.x)
- Reference Issues: https://github.com/vuejs/vue-router/issues/2611,
- Implementation PR: (leave this empty)

# Summary

- Remove `tag` prop
- Remove `event` prop
- Stop automatically assigning click events to inner anchors
- Add a scoped-slot API

# Basic example

```vue
<router-link to="/">
  <Icon>home</Icon> Home
</router-link>
```

# Motivation

Current implementation of Router Link has many limitations:

- Active state customization is not complete ([#2611](https://github.com/vuejs/vue-router/issues/2611)
- Cannot be integrated with custom components ([#2021](https://github.com/vuejs/vue-router/issues/2021))
- click event cannot be prevented (through `@click.prevent` nor through a `disabled` attribute [#2098](https://github.com/vuejs/vue-router/pull/2098))

The idea of this RFC is to solve those issues by providing a scoped slot that allow app developers to easily extend Links based on their applications and to allow library authors to provide an even easier integration with Vue Router.

# Detailed design

## Slot with content

A simple use case would be slot with content (no nested anchors or buttons)

```vue
<router-link to="/">
  <Icon>home</Icon> Home
</router-link>
```

This implementation would:

- generate an anchor (`a`) element and apply the corresponding properties:
  - `href` with the destination
  - `class` with `router-link-active` or `router-link-exact-active` (can be changed through prop or global option)
  - click listener to trigger navigation through `router.push` or `router.replace` with a `event.preventDefault`
- Put anything passed as the children of the anchor
- Pass down any attributes that aren't props to the `a` element

**Breaking changes**:

- no longer accept a `tag` prop -> use the scoped slot instead (see the point below)
- no longer accepts `event` -> use the scoped slot instead
- no longer works as a wrapper automatically looking for the first `a` inside -> use the scoped slot instead

## Custom `tag` prop

I am not sure about keeping the `tag` prop if it can be replaced which a scoped slot because it wouldn't handle custom components and except for very simple cases, we will likely use custom UI components instead of the basics ones:

```vue
<router-link to="/" tag="button">
  <Icon>home</Icon><span class="xs-hidden">Home</span>
</router-link>
```

is equivalent to

```vue
<router-link to="/" v-slot="{ navigate, classes }">
  <button role="link" @click="navigate" :class="classes">
    <Icon>home</Icon><span class="xs-hidden">Home</span>
  </button>
</router-link>
```

(see below for explanation about the attributes passed to the scoped-slot)

## Scoped slot

A scoped slot would get access to every bit of information needed to provide a custom integration and allows applying the active classes, click listener, links, etc at any level. This would allow a better integration with frameworks like Bootstrap (https://getbootstrap.com/docs/4.3/components/navbar/). The idea would be to create a Vue component to avoid the boilerplate like bootstrap-vue does (https://bootstrap-vue.js.org/docs/components/navbar/#navbar)

```vue
<router-link to="/" v-slot="{ href, navigate, classes }">
  <li :class="classes">
    <a :href="href" @click.prevent="navigate">
      <Icon>home</Icon><span class="xs-hidden">Home</span>
    </a>
  </li>
</router-link>
```

### Accessible variables

The variables fall into multiple categories. In documenation, they should be split per usage as [@Akryum pointed out](https://github.com/vuejs/rfcs/pull/34#issuecomment-491381810). This is important to overcome the fact that there is a big amount of available information

#### Essential

These are what you will likely *always* use

- `href`: resolved relative url to be added to an anchor tag (is this necessary as it's an alias to `route.fullPath`)
- `route`: resolved normalized route location from the `to` (same shape as `$route`)
- `navigate`: function to trigger navigation (usually attached to a click)
- `isActive`: true whenever `router-link-active` is applied. This is when the _path_ section of the href (without the query and hash) is included (as in `currentLocation.path.startsWith(location.path)`). Can be modified by `exact` prop
- `isExactActive`: true whenever `router-link-exact-active` is aplied. Only applies when the _fullPath_ is matched (including query and hash) (as in `currentLocation.fullPath === location.fullPath`). Can be modified by `exact` prop.

#### Convenience

These are here for convenience and could be replicated in user land

- `classes`: classes to be applied to an element. Will contain `router-link-active` and/or `router-link-exact-active`
- `isSameFullPath`: true if the fullPath
- `isSubPath`: true if the path section of the link is included in current route as in `$route.path.includes(route.path)`
- `isSamePath`: true if the path section of the link matches current route (equality comparison between strings)
- `isSameQuery`: true if the query of the link matches current query (custom equality comparison between objects)
- `isSameHash`: true if the hash of the link matches current hash (equality comparison between strings)
- `isDescendant`: true if the route record matched by the `to` location is a descendant of the record matched by the current route (as in `children` (works for nested `children` as well))
- `isAscendant`: true if the route record matched by the `to` location is an ascendant of the record matched by the current route (as in current route being in `children`(works for nested routes as well))

# Drawbacks

- Whereas it's possible to keep existing behaviour working and only expose a new behaviour with scoped slots, it will still prevent us from fixing existing issues with current implementation. That's why there are some breaking changes, to make things more consistent.
- Too many scoped props, only `isSameQuery`, `isDescendant` and `isAscendant` are difficult to replicate in userland.

# Alternatives

- Keeping `event` prop for convienience

# Adoption strategy

- Document new slot behaviour based on examples
- Deprecate `tag` and `event` with a message and link to documentation the remove in v4

# Unresolved questions

- Do we need other information in the scoped slot?
