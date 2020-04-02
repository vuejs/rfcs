- Start Date: 2019-11-05
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- `v-on` listeners used on a component will fallthrough and be registered as native listeners on the child component root. `.native` modifier is no longer needed.

- `inheritAttrs: false` now affects `class` and `style`.

- `this.$attrs` now contains everything passed to the component minus those explicitly declared as props, including `class`, `style`, and `v-on` listeners. `this.$listeners` is removed.

- Functional components attribute fallthrough behavior adjusted:
  - With explicit `props` declaration: full fallthrough like stateful components.
  - With no `props` declaration: only fallthrough for `class`, `style` and `v-on` listeners.

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

In 3.x, we are also introducing Fragments (multiple root nodes in a component template), which require additional considerations on the behavior.

# Detailed design

## `v-on` Listener Fallthrough

With the following usage:

```html
<MyButton @click="hello" />
```

- In v2, the `@click` will only register a component custom event listener. To attach a native listener to the root of `MyButton`, `@click.native` is needed.

- In v3, the `@click` listener will fallthrough and register a native click listener on the root of `MyButton`. This means component authors no longer need to proxy native events to custom events in order to support `v-on` usage without the `.native` modifier. In fact, the `.native` modifier will be removed altogether.

  Note this may result in unnecessary registration of native event listeners when the user is only listening to component custom events, which we discuss below in [Unresolved Questions](#unresolved-questions).

## Explicitly Controlling the Fallthrough

### `inheritAttrs: false`

With `inheritAttrs: false`, the implicit fallthrough is disabled. The component can either choose to intentionally ignore all extraneous attrs, or explicitly control where the attrs should be applied via `v-bind="$attrs"`:

``` html
<div class="wrapper">
  <!-- apply attrs to an inner element instead of root -->
  <input v-bind="$attrs">
</div>
```

`this.$attrs` (and `context.attrs` for `setup()` and functional components) now contains all attributes passed to the component (as long as it is not declared as props). This includes `class`, `style`, normal attributes and `v-on` listeners. This is based on the flat props structure proposed in [Render Function API Change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md#flat-vnode-props-format).

`v-on` listeners are included in `$attrs` as `onXXX` props. For example, `@click` will result in an `onClick` prop in `$attrs`. If the user wants to handle attributes and listeners separately, it can be done with simple helper functions that separate props that start with `on` from those that don't.

### Multiple Root / Fragment Components

In Vue 3, components can have multiple root elements (i.e. fragment root). In such cases, an automatic merge cannot be performed. The user will be responsible for spreading the attrs to the desired element:

``` html
<template>
  <span>hello</span>
  <div v-bind="$attrs">main element</div>
</template>
```

If `$attrs` is non-empty, and the user did not perform an explicit spread (checked by access to `this.$attrs` during render), a runtime warning will be emitted. The component should either bind `$attrs` to an element, or explicitly suppress the warning with `inheritAttrs: false`.

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

If returning the render function from `setup`, the attrs object is exposed on the setup context:

``` js
import { mergeProps } from 'vue'

export default {
  props: { /* ... */ },
  inheritAttrs: false,
  setup(props, { attrs }) {
    return () => {
      return h('div', mergeProps({ class: 'foo' }, attrs))
    }
  }
}
```

Note the `attrs` object is updated before every render, so it's ok to destructure it.

## Functional Components

In 2.x, functional components do not support automatic attribute fallthrough and require manual props merging.

In v3, functional components use a different syntax: they are now declared as plain functions (as specified in [Render Function API Change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md#functional-component-signature)).

### With Explicit Props Declaration

A functional component with `props` declaration will have the same automatic fallthrough behavior as stateful components. It can also explicitly control the attrs with `inheritAttrs: false`:

``` js
const Func = (props, { attrs }) => {
  return h('div', mergeProps({ class: 'foo' }, attrs), 'hello')
}

Func.props = { /*...*/ }
Func.inheritAttrs = false
```

### With Optional Props Declaration

v3 functional components also support [Optional Props Declaration](#TODO). When a functional component has no `props` option defined, it receives all attributes passed by the parent as its `props`:

``` js
const Foo = props => h('div', { class: 'foo' }, props.msg)
```

When a functional component is leveraging optional props declaration, there is only implicit fallthrough for `class`, `style`, and `v-on` listeners.

The reason for `class`, `style` and `v-on` listeners to be whitelisted is because:

- They cover the most common use cases for attribute fallthrough.
- They have close to no risk of clashing with prop names.
- They require special merge logic instead of simple overwrites, so handling them implicitly yields more convenience value.

If a functional component with optional props declaration needs to support full attribute fallthrough, it needs to declare `inheritAttrs: false`, pick the desired attrs from `props`, and merge it to the root element:

``` js
// destructure props, and use rest spread to grab unused ones as attrs.
const Func = ({ msg, ...attrs }) => {
  return h('div', mergeProps({ class: 'foo' }, attrs), msg)
}
Func.inheritAttrs = false
```

## API Deprecations

- `.native` modifier for v-on will be removed.
- `this.$listeners` will be removed.

# Adoption strategy

- Deprecations can be supported in the compat build:

  - `.native` modifier will be a no-op and emit a warning during template compilation.
  - `this.$listeners` can be supported with a runtime warning.

- There could technically be cases where the user relies on the 2.x behavior where `inheritAttrs: false` does not affect `class` and `style`, but it should be very rare. We will have a dedicated item in the migration guide / helper to remind the developer to check for such cases.

- Since functional components uses a new syntax, they will likely require manual upgrades. We should have a dedicated section for functional components in the migration guide.

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
