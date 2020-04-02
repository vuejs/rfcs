- Start Date: 2019-04-29
- Target Major Version: Vue (2.x / 3.x) Vue Router (3.x / 4.x)
- Reference Issues: https://github.com/vuejs/vue-router/issues/2611,
- Implementation PR: Implemented in both [v3.x](https://github.com/vuejs/vue-router/commit/e289ddee99fcc3129e65485e32f394c1308bb98b) and v4.x

# Summary

- Remove `tag` prop
- Remove `event` prop
- Stop automatically assigning click events to inner anchors
- Add a scoped-slot API
- Add a `custom` prop to fully customize `router-link`'s rendering

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
  - `class` with `router-link-active` and/or `router-link-exact-active` (can be changed through prop or global option)
  - click listener to trigger navigation through `router.push` or `router.replace` with a `event.preventDefault` (except when the link is clicked using a modifier like <kbd>⌘</kbd> or <kbd>Ctrl</kbd>)
- Put anything passed as the children of the anchor
- Pass down any attributes that aren't props to the `a` element

**Breaking changes**:

- no longer accept a `tag` prop -> use the scoped slot instead (see the point below)
- no longer accepts `event` -> use the scoped slot instead
- no longer works as a wrapper automatically looking for the first `a` inside -> use the scoped slot instead

## Scoped slot

A scoped slot would get access to every bit of information needed to provide a custom integration and allows applying the active classes, click listener, links, etc at any level. This would allow a better integration with UI frameworks like Bootstrap (https://getbootstrap.com/docs/4.3/components/navbar/). The idea would be to create a Vue component to avoid the boilerplate like bootstrap-vue does (https://bootstrap-vue.js.org/docs/components/navbar/#navbar)

```vue
<router-link to="/" custom v-slot="{ href, navigate, isActive }">
  <li :class="{ 'active': isActive }">
    <a :href="href" @click="navigate">
      <Icon>home</Icon><span class="xs-hidden">Home</span>
    </a>
  </li>
</router-link>
```

The `custom` prop is necessary to take full control over `router-link`'s rendering: not rendering a wrapping `a` element.

**Why is a `custom` prop necessary**: in Vue 3, scoped slots and regular slots cannot be differentiated from each other, which means vue router is unable to make the difference between these 3 cases:

```vue
<router-link to="/" v-slot="{ href, navigate, isActive }"></router-link>
<router-link to="/" v-slot></router-link>
<router-link to="/">Some Link</router-link>
```

In all three cases we need to render the slot content but `router-link` needs to know if it has to render a wrapping `a` element. In Vue 2, we are able to do so by checking `$scopedSlots` but in Vue 3, only `slots` exists. This means that the behavior is slightly different in Vue Router v3 and Vue Router v4:

- In v3, the `custom` prop is required (see [Adoption strategy](#adoption-strategy)) alongside `v-slot`. `router-link` will not wrap the slot content with an `a` element.
- In v4, the `custom` prop is **not** required alongside `v-slot`. It controls whether `router-link` should wrap its slot content with an `a` element or not:
  ```vue
  <router-link to="/" v-slot="{ href }">
  <router-link to="/" custom v-slot="{ href, navigate }">
    <a :href="href" @click="navigate">{{ href }}</a>
  </router-link>
  <!-- both render the same -->
  <a href="/">/</a>
  <a href="/">/</a>
  ```

### Accessible variables

The slot should provide values that are computed inside `router-link`:

- `href`: resolved relative url to be added to an anchor tag (contains the base if provided while route.fullPath doesn't)
- `route`: resolved normalized route location from the `to` (same shape as `$route`)
- `navigate`: function to trigger navigation (usually attached to a click). Also calls `preventDefault` if the click is directly pressed.
- `isActive`: true whenever `router-link-active` is applied. Can be modified by `exact` prop
- `isExactActive`: true whenever `router-link-exact-active` is aplied. Can be modified by `exact` prop.

## The removal of the `tag` prop

The `tag` prop can be replaced which a scoped slot and make the code clearer while not being exposed to any caveat. Its removal will also lighten the vue-router library.

```vue
<router-link to="/" tag="button">
  <Icon>home</Icon><span class="xs-hidden">Home</span>
</router-link>
```

is equivalent to

```vue
<router-link to="/" custom v-slot="{ navigate, isActive, isExactActive }">
  <button role="link" @click="navigate" :class="{ active: isActive, 'exact-active': isExactActive }">
    <Icon>home</Icon><span class="xs-hidden">Home</span>
  </button>
</router-link>
```

(see above for explanation about the attributes passed to the scoped-slot)

# Drawbacks

- Whereas it's possible to keep existing behaviour working and only expose a new behaviour with scoped slots, it will still prevent us from fixing existing issues with current implementation. That's why there are some breaking changes, to make things more consistent.
- No access to the default `router-link` classes like `router-link-active` and `router-link-exact-active`.

# Alternatives

- Keeping `event` prop for convienience
- Use a different named slot instead of a `prop`:

  ```vue
  <router-link #custom="{ href }">
    <a :href="href"></a>
  </router-link>

  <router-link v-slot:custom="{ href }">
    <a :href="href"></a>
  </router-link>

  <router-link custom v-slot="{ href }">
    <a :href="href"></a>
  </router-link>
  ```

  The adoption strategy in this case would be similar but the warning would tell the user to use a different slot instead of a prop named `custom`

- Create a new component like `router-link-custom` to differentiate the behavior. This solution is however heavier (in terms of size) than a prop or a different named slot.

# Adoption strategy

- Document new slot behaviour based on examples
- Deprecate `tag` and `event` with a message in v3 and link to documentation, then remove in v4
- In v3, if no `custom` prop is provided when using a scoped slot, warn the user to use the `custom` prop
