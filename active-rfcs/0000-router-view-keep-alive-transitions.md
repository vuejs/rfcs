- Start Date: 2020-04-20
- Target Major Version: Vue Router 4.x
- Reference Issues:
- Implementation PR:

# Summary

Given the changes to functional components in Vue 3, `KeepAlive` and `Transition` usage combined with `RouterView` is no longer possible by simply wrapping `RouterView` with `KeepAlive`/`Transition`. Instead we need a way to directly provide the component rendered by `RouterView` to those components:

```vue
<router-view v-slot="{ Component }">
  <transition :name="transitionName" mode="out-in">
    <component :is="Component"></component>
  </transition>
</router-view>
```

# Motivation

As described at https://github.com/vuejs/vue-next/issues/906#issuecomment-611080663, we need a new API to allow the usage of `KeepAlive` and other components accepting slots with `RouterView`. In order to implement such behavior we need to be able to **directly** wrap the component rendered by `RouterView`. The only way to do so by having access to the component rendered by `RouterView` and the _props_ passed to it.

# Detailed design

We can achieve this level of control through a slot:

```vue
<router-view v-slot="{ Component, route }">
  <component :is="Component" v-bind="route.params"></component>
</router-view>
```

In the example, `Component` is the _component definition_ that can be passed to the function `h` or to the `is` prop of `component`.

When defining a `props: true` option in the route definition:

```js
createRouter({
  routes: [{ path: '/users/:id', component: User, props: true }],
})
```

The `router-view` will automatically add the `id` prop to the rendered component with the value of the param named `id`. Note you can also do `v-bind="route.params"` like shown in the example above **instead** of using `props: true`.

## No match case

When the current location isn't matched by any record registered by the Router, the `matched` array of a `RouteLocation` is empty and, by default, when no _slot_ is provided, it renders nothing. When we provide a _slot_, we can decide of what to display, whether we want to display a _not found_ page or we want the default behavior, we are able to do it. `Component` will be falsy if there is no component to render:

```vue
<router-view v-slot="{ Component }">
  <component :is="Component"></component>
  <div v-else>Not Found</div>
</router-view>
```

Note that this behavior is redundant with the _catch all_ route (`path: '/:pathMatch(.*)`) when it comes to displaying a not found page.

## `v-slot` properties

- `Component`: _Component_ that can be passed to the function `h` or to the `is` prop of `component`.
- `route`: Normalized Route location rendered by `RouterView`. Same as `$route`.

# Drawbacks

# Wrapping `RouterView` with `Transition` or `KeepAlive`

If the user accidentally wraps `RouterView` with `Transition` or is migrating their application to Vue 3, we could issue a warning pointing to the documentation (this RFC in the meantime) and hinting them to use the `v-slot` api.

# Alternatives

- Using a function like `useView` that would return `Component` and `attrs` removing the use of `v-slot`.

# Adoption strategy

- A codemod could rewrite v3 to v4.
