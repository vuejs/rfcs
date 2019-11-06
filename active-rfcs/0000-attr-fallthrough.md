- Start Date: 2019-11-05
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- Make the attrs fallthrough behavior more consistent;
- Make it easier to pass all extraneous attrs to child elements / components.

# Motivation

In Vue 2.x, components have an implicit attrs fallthrough behavior. Any prop passed to a component, but is not declared as a prop by the component, is considered an "extraneous attribute". In 2.x, these extraneous attributes are exposed in `this.$attrs` and implicitly applied to the component's root node. This behavior can be disabled with `inheritAttrs: false`, where the user expects to explicit control where the attrs should be applied.

There are a number of inconsistencies and issues in the 2.x behavior:

- `inheritAttrs: false` does not affect `class` and `style`.

- `class`, `style`, `v-on` listeners and custom directives are not included in `$attrs`, making it cumbersome for a higher-order component (HOC) to properly pass everything down to a nested child component.

- Functional components have no implicit attrs fallthrough at all.

In 3.x, the need for "spreading extraneous attrs" also becomes more prominent due to the ability for components to render multiple root nodes (fragments). This RFC seeks to address these problems.

# Detailed design

`this.$attrs` now contains **everything** passed to the component except those that are declared as props. **This includes `class`, `style`, `v-on` listeners (as `onXXX` props), and custom directives (as `onVnodeXXX` props)**. (This is based on flat props structure as proposed in [Render Function API Change](https://github.com/vuejs/rfcs/blob/render-fn-api-change/active-rfcs/0000-render-function-api-change.md#flat-vnode-props-format)). As a result of this:

- `.native` modifier for `v-on` will be removed.

- `this.$listeners` will be removed.

When the component returns a single root node, `this.$attrs` will be implicitly merged into the root node's props. This is the same as 2.x, except it will now include all the props that were not previously in `this.$attrs`, as discussed above.

If the component receives extraneous attrs, but returns multiple root nodes (a fragment), an automatic merge cannot be performed. If the user did not perform an explicit spread (checked by access to `this.$attrs` during render), a runtime warning will be emitted. The component should either pick an element to apply the attrs to (via `v-bind="$attrs"`), or explicitly suppress the warning with `inheritAttrs: false`.

## `inheritAttrs: false`

With `inheritAttrs: false`, the component can either choose to intentionally ignore all extraneous attrs, or explicitly control where the attrs should be applied via `v-bind="$attrs"`:

``` html
<div class="wrapper">
  <!-- apply attrs to an inner element instead of root -->
  <input v-bind="$attrs">
</div>
```

In 2.x, this option does not affect `class` and `style` - they will be implicitly merged on root in all cases for stateful components - but in 3.0 this special case is removed: `class` and `style` will be part of `$attrs` just like everything else.

## Merging Attrs in Render Functions

In manual render functions, it may seem convenient to just use a spread:

``` js
export default {
  props: { /* ... */ },
  inheritAttrs: false,
  render() {
    return h('div', { class: 'foo', ...this.$attrs })
  }
}
```

However, this will cause attrs to overwrite whatever existing props of the same name. For example, there the local `class` may be overwritten when we probably want to merge the classes instead. Vue provides a `mergeProps` helper that handles the merging of `class`, `style` and `onXXX` listeners:

``` js
import { mergeProps } from 'vue'

export default {
  props: { /* ... */ },
  inheritAttrs: false,
  render() {
    return h('div', mergeProps({ class: 'foo' }, this.$attrs))
  }
}
```

This is also what `v-bind` uses internally.

## Consistency between Functional and Stateful Components

Functional components will now share the exact same behavior with Stateful components. The extraneous attrs is passed via the second context argument (as specified in [Render Function API Change](https://github.com/vuejs/rfcs/blob/render-fn-api-change/active-rfcs/0000-render-function-api-change.md#functional-component-signature)):

``` js
const Func = (props, { attrs }) => {
  return h('div', mergeProps({ id: 'x' }, attrs), props.msg)
}

Func.props = { /* ... */ }
```

## Components with no Props Declaration

Note that for components without props declaration (see [Optional Props Declaration](https://github.com/vuejs/rfcs/pull/25)), there will be no implicit attrs handling of any kind, because everything passed in is considered a prop and there will be no "extraneous" attrs. A component without props declaration (mostly functional components) is responsible for explicitly passing down necessary props. This can be easily done with object rest spread:

``` js
const Func = ({ msg, ...rest }) => {
  return h('div', mergeProps({ id: 'x' }, rest), [
    h('span', msg)
  ])
}
```

# Drawbacks

For existing components using `inheritAttrs: false` this will be a breaking change. However, the upgrade should lead to simpler code.

# Alternatives

N/A

# Adoption strategy

- Migration guide for existing components using `inheritAttrs: false`.
- Rework documentation regarding `$attrs`.

# Unresolved questions

## Removing Unwanted Listeners

With flat VNode data and the removal of `.native` modifier, all listeners are passed down to the child component as `onXXX` functions:

``` html
<foo @click="foo" @custom="bar" />
```

compiles to:

``` js
h(foo, {
  onClick: foo,
  onCustom: bar
})
```

When spreading `$attrs` with `v-bind`, all parent listeners are applied to the target element as native DOM listeners. The problem is that these same listeners can also be triggered by custom events - in the above example, both a native click event and a custom one emitted by `this.$emit('click')` in the child will trigger the parent's `foo` handler. This may lead to unwanted behavior.

Props do not suffer from this problem because declared props are removed from `$attrs`. Therefore we should have a similar way to "declare" emitted events from a component. There is currently [an open RFC for it](https://github.com/vuejs/rfcs/pull/16) by @niko278.

Event listeners for explicitly declared events will be removed from `$attrs` and can only be triggered by custom events emitted by the component via `this.$emit`.
