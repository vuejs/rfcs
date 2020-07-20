- Start Date: 2019-04-09
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- Re-design custom directive API so that it better aligns with component lifecycle

- Custom directives usage on components will follow the same rules as discussed in the [Attribute Fallthrough Behavior RFC](https://github.com/vuejs/rfcs/pull/26). It will be controlled by the child component via `v-bind="$attrs"`.

# Basic example

## Before

``` js
const MyDirective = {
  bind(el, binding, vnode, prevVnode) {},
  inserted() {},
  update() {},
  componentUpdated() {},
  unbind() {}
}
```

## After

``` js
const MyDirective = {
  beforeMount(el, binding, vnode, prevVnode) {},
  mounted() {},
  beforeUpdate() {},
  updated() {},
  beforeUnmount() {}, // new
  unmounted() {}
}
```

# Motivation

Make custom directive hook names more consistent with the component lifecycle.

# Detailed design

## Hooks Renaming

Existing hooks are renamed to map better to the component lifecycle, with some timing adjustments. Arguments passed to the hooks remain unchanged.

- `bind` -> `beforeMount`
- `inserted` -> `mounted`
- `beforeUpdate` *new, called before the element itself is updated*
- ~~`update`~~ *removed, use `updated` instead*
- `componentUpdated` -> `updated`
- `beforeUnmount` *new*
- `unbind` -> `unmounted`

## Usage on Components

In 3.0, with fragments support, components can potentially have more than one root nodes. This creates an issue when a custom directive is used on a component with multiple root nodes.

To explain the details of how custom directives will work on components in 3.0, we need to first understand how custom directives are compiled in 3.0. For a directive like this:

``` html
<div v-foo="bar"></div>
```

Will roughly compile into this:

``` js
const vFoo = resolveDirective('foo')

return withDirectives(h('div'), [
  [vFoo, bar]
])
```

Where `vFoo` will be the directive object written by the user, which contains hooks like `mounted` and `updated`.

`withDirectives` returns a cloned VNode with the user hooks wrapped and injected as vnode lifecycle hooks (see [Render Function API Changes](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md#special-reserved-props) for more details):

``` js
{
  onVnodeMounted(vnode) {
    // call vFoo.mounted(...)
  }
}
```

**As a result, custom directives are fully included as part of a VNode's data. When a custom directive is used on a component, these `onVnodeXXX` hooks are passed down to the component as extraneous props and end up in `this.$attrs`.**

This also means it's possible to directly hook into an element's lifecycle like this in the template, which can be handy when a custom directive is too involved:

``` html
<div @vnodeMounted="myHook" />
```

This is consistent with the attribute fallthrough behavior discussed in [vuejs/rfcs#26](https://github.com/vuejs/rfcs/pull/26). So, the rule for custom directives on a component will be the same as other extraneous attributes: it is up to the child component to decide where and whether to apply it. When the child component uses `v-bind="$attrs"` on an inner element, it will apply any custom directives used on it as well.

# Drawbacks

N/A

# Alternatives

N/A

# Adoption strategy

- The renaming should be easy to support in the compat build
- Codemod should also be straightforward
- For directives used on components, the warning on unused `$attrs` as discussed in [Attribute Fallthrough Behavior](https://github.com/vuejs/rfcs/pull/26) should apply as well.

# Unresolved questions

N/A
