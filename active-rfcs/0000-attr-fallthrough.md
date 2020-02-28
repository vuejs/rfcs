- Start Date: 2019-11-05
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- Implicit fallthrough now by default only applies for a whitelist of attributes (`class`, `style`, event listeners, and a11y attributes).

- Implicit fallthrough now works consistently for both stateful and functional components (as long as the component has a single root node).

- `this.$attrs` now contains everything passed to the component minus those explicitly declared as props, including `class`, `style`, and listeners. `this.$listeners` is removed.

# Motivation

In Vue 2.x, components have an implicit attributes fallthrough behavior. Any attribute passed to a component that is not declared as a prop by the component, is considered an **extraneous attribute**. Fore example:

``` html
<MyComp id="foo"/>
```

If `MyComp` didn't declare a prop named `id`, then the `id` is considered an extraneous attribute and will implicitly be applied to the root node of `MyComp`.

This behavior is very convenient when tweaking layout styling between parent and child (by passing on `class` and `style`), or applying a11y attributes to child components.

This behavior can be disabled with `inheritAttrs: false`, where the user expects to explicitly control where the attributes should be applied. These extraneous attributes are exposed in an instance property: `this.$attrs`.

There are a number of inconsistencies and issues in the 2.x behavior:

- `inheritAttrs: false` does not affect `class` and `style`.

- Implicit fallthrough does not apply for event listeners, leading to the need for `.native` modifier if the user wish to add a native event listener to the child component root.

- `class`, `style` and `v-on` listeners are not included in `$attrs`, making it cumbersome for a higher-order component (HOC) to properly pass everything down to a nested child component.

- Functional components have no implicit attrs fallthrough behavior.

In 3.x, we are also introducing additional features such as [Optional Props Declaration](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0010-optional-props-declaration.md) and Fragments (multiple root nodes in a component template), both of which require additional considerations on the behavior.

# Detailed design

## Implicit Fallthrough Whitelist

Implicit fallthrough now by default only affects a whitelist of attributes:

- `class`
- `style`
- `v-on` event listeners (as `onXXX` props)
- Accessibility attributes: `aria-xxx` and `role`.
- Data attributes: `data-xxx`.

To qualify for the whitelist, an attribute must:

- have a common use case that relies on fallthrough
- have a low chance of being used as a component prop (e.g. `id` is not a good candidate)
- apply to all element types (e.g. `alt` is not included because it only applies for `img`).

## Why Is There a Whitelist?

We introduced [Optional Props Declaration](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0010-optional-props-declaration.md) in a previous RFC. However, optional props won't really be practical if the implicit fallthrough applies to all attributes. Consider the following example (using 3.x functional components for brevity):

``` js
const Foo = props => h('div', props.msg)
```

Here the component intends to use `msg` as a component prop, however with implicit fallthrough, the root `div` will also render `msg` as a fallthrough attribute. To avoid the undesired fallthrough, the component would still need to explicitly declare props.

Another option is disabling the implicit fallthrough if the component has no props declared. However, not only is this inconsistent and confusing, it's also a major breaking change. In 2.x, a component without props simply means it does not expect any props and the user would expect the implicit fallthrough to be applied.

The whitelist is a middle ground where:

- Users can still rely on the convenience of implicit fallthrough for the most common cases.
- Users can enjoy more succinct component authoring without having to always declare props (especially for functional components).

## Single Root vs. Fragment Root

When the component returns a single root node, `this.$attrs` will be implicitly merged into the root node's props. This is the same as 2.x, except it will now include all the props that were not previously in `this.$attrs`, as discussed above.

If the component receives extraneous attrs, but returns multiple root nodes (a fragment), an automatic merge cannot be performed. If the user did not perform an explicit spread (checked by access to `this.$attrs` during render), a runtime warning will be emitted. The component should either pick an element to apply the attrs to (via `v-bind="$attrs"`, see section below), or explicitly suppress the warning with `inheritAttrs: false`.

## Explicitly Controlling the Fallthrough

`this.$attrs` (and `context.attrs` for `setup()` and functional components) now contains all attributes passed to the component (as long as it is not declared as props). This includes `class`, `style`, normal attributes and `v-on` listeners (as `onXXX` props). This is based on the flat props structure proposed in [Render Function API Change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md#flat-vnode-props-format).

In case the user wants to let an attribute that is not in the whitelist fallthrough, e.g. `id`, it can be added individually as:

``` html
<div :id="$attrs.id"></div>
```

### `inheritAttrs: false`

With `inheritAttrs: false`, the implicit fallthrough is disabled. The component can either choose to intentionally ignore all extraneous attrs, or explicitly control where the attrs should be applied via `v-bind="$attrs"`:

``` html
<div class="wrapper">
  <!-- apply attrs to an inner element instead of root -->
  <input v-bind="$attrs">
</div>
```

Note `$attrs` will also include attributes not in the implicit whitelist. So this pattern can also be used to get around the whitelist and force all attributes to fallthrough.

> In 2.x, `inheritAttrs` does not affect `class` and `style` - they will still be merged onto the root element. With this RFC, this special case is removed: `class` and `style` will be part of `$attrs` just like everything else.

### In Render Functions

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

Functional components will now share the exact same behavior with Stateful components. The extraneous attrs is passed via the second context argument (as specified in [Render Function API Change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md#functional-component-signature)):

``` js
const Func = (props, { attrs }) => {
  return h('div', mergeProps({ id: 'x' }, attrs), props.msg)
}

Func.props = { /* ... */ }
```

## API Deprecations

- `.native` modifier for v-on will be removed. `v-on` listeners attached to a component now implicitly fallthrough to the child component root, and is also included in `$attrs` in case of manual inheritance.

- `this.$listeners` will be removed for the same reason.

# Drawbacks

- Attributes not in the whitelist will now need to be explicitly applied in the child component. However, such cases should be rare and we can detect and warn the presence of such attributes in the compatibility build.

# Adoption strategy

There are two types of 2.x components affected by these changes:

1. Component using `inheritAttrs: false`. We can detect and warn usage of `this.$listeners` in the compatibility build.

    There could technically be cases where the user relies on the 2.x `class` and `style` behavior with `inheritAttrs: false`, but it should be very rare. We will have a dedicated item in the migration guide / helper to remind the developer to check for such cases.

2. Components relying on fallthrough of attributes not in the whitelist. This should be a relatively rare case and can be detected and warned against in the compatibility build

Additionally:

- `.native` modifier will be a no-op and emit a migration warning.

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

Props do not suffer from this problem because declared props are removed from `$attrs`. Therefore we should have a similar way to "declare" emitted events from a component. Event listeners for explicitly declared events will be removed from `$attrs` and can only be triggered by custom events emitted by the component via `this.$emit`. There is currently [an open RFC for it](https://github.com/vuejs/rfcs/pull/16) by @niko278. It is complementary to this RFC but does not affect the design of this RFC, so we can leave it for consideration at a later stage, even after Vue 3 release.
