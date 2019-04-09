- Start Date: 2019-04-08
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- Disable implicit attribute fall-through to child component root element

- Remove `inheritAttrs` option

# Basic example

To replicate 2.x behavior in templates:

``` html
<div v-bind="$attrs">hi</div>
```

In render function:

``` js
import { h } from 'vue'

export default {
  render() {
    return h('div', this.$attrs, 'hi')
  }
}
```

# Motivation

In 2.x, the current attribute fallthrough behavior is quite implicit:

- `class` and `style` used on a child component are implicitly applied to the component's root element. It is also automatically merged with `class` and `style` bindings on that element in the child component template.

  - However, this behavior is not consistent in functional components because functional components may return multiple root nodes.

  - With 3.0 supporting fragments and therefore multiple root nodes for all components, this becomes even more problematic. The implicit behavior can suddenly fail when the child component changes from single-root to multi-root.

- attributes passed to a component that are not declared by the component as props are also implicitly applied to the component root element.

  - Again, in functional components this needs explicit application, and would be inconsistent for 3.0 components with multiple root nodes.

  - `this.$attrs` only contains attributes, but excludes `class` and `style`; `v-on` listeners are contained in a separate `this.$listeners` object. There is also the `.native` modifier. The combination of `inheritAttrs`, `.native`, `$attrs` and `$listeners` makes props passing in higher-order components unnecessarily complex. The new behavior makes it much more straightforward: spreading $attrs means "pass everything that I don't care about down to this element/component".

  - `class` and `style` are always automatically merged, and are not affected by `inheritAttrs`.

The fallthrough behavior has already been inconsistent between stateful components and functional components in 2.x. With the introduction of fragments (the ability for a component to have multiple root nodes) in 3.0, the fallthrough behavior becomes even more unreliable for component consumers. The implicit behavior is convenient in cases where it works, but can be confusing in cases where it doesn't.

In 3.0, we are planning to make attribute fallthrough an explicit decision of component authors. Whether a component accepts additional attributes becomes part of the component's API contract. We believe overall this should result in a simpler, more explicit and more consistent API.

# Detailed design

- `inheritAttrs` option will be removed.

- `.native` modifier for `v-on` will be removed.

- Non-prop attributes no longer automatically fallthrough to the root element of the child component (including `class` and `style`). This is the same for both stateful and functional components.

  This means that with the following usage:

  ``` js
  const Child = {
    props: ['foo'],
    template: `<div>{{ foo }}</div>`
  }

  const Parent = {
    components: { Child },
    template: `<child foo="1" bar="2" class="bar"/>`
  }
  ```

  Both `bar="2"` AND `class="bar"` on `<child>` will be ignored.
 
- `this.$listeners` will be removed.

- `this.$attrs` now contains **everything** passed to the component except those that are declared as props or custom events. **This includes `class`, `style`, `v-on` listeners (as `onXXX` properties)**. The object will be flat (no nesting) - this is possible thanks to the new flat VNode data structure (discussed in [Render Function API Change](https://github.com/vuejs/rfcs/pull/28)).

  To explicitly inherit additional attributes passed by the parent, the child component should apply it with `v-bind`:

  ``` js
  const Child = {
    props: ['foo'],
    template: `<div v-bind="$attrs">{{ foo }}</div>`
  }
  ```

  This also applies when the child component needs to apply `$attrs` to a non-root element, or has multiple root nodes:

  ``` js
  const ChildWithNestedRoot = {
    props: ['foo'],
    template: `
      <label>
        {{ foo }}
        <input v-bind="$attrs">
      </label>
    `
  }

  const ChildWithMultipleRoot = {
    props: ['foo'],
    template: `
      <label :for="$attrs.id">{{ foo }}</label>
      <input v-bind="$attrs">
    `
  }
  ```

  In render functions, if simple overwrite is acceptable, `$attrs` can be merged using object spread. But in most cases, special handling is required (e.g. for `class`, `style` and `onXXX` listeners). Therefore a `mergeData` helper will be provided. It handles the proper merging of VNode data:

  ``` js
  import { h, mergeData } from 'vue'

  const Child = {
    render() {
      return h(InnerComponent, mergeData({ foo: 'bar' }, this.$attrs))
    }
  }
  ```

  Inside render functions, the user also has the full flexibility to pluck / omit any props from `$attrs` using 3rd party helpers, e.g. lodash.

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

# Drawbacks

- Fallthrough behavior is now disabled by default and is controlled by the component author. If the component is intentionally "closed" there's no way for the consumer to change that. This may cause some inconvenience for users accustomed to the old behavior, especially when using `class` and `style` for styling purposes, but it is the more "correct" behavior when it comes to component responsibilities and boundaries. Styling use cases can be easily worked around with by wrapping the component in a wrapper element. In fact, this should be the best practice in 3.0 because the child component may or may not have multiple root nodes.

- For accessibility reasons, it should be a best practice for components that are shipped as libraries to always spread `$attrs` so that any `aria-x` attributes can fallthrough. However this is a straightforward / mechanical code change, and is more of an educational issue. We could make it common knowledge by emphasizing this in all our information channels.

# Alternatives

N/A

# Adoption strategy

## Documentation

This RFC discusses the problem by starting with the 2.x implementation details with a lot of history baggage so it can seem a bit complex. However if we were to document the behavior for a new user, the concept is much simpler in comparison:

- For a component without explicit props and events declarations, everything passed to it from the parent ends up in `$attrs`.

- If a component declares explicit props, they are removed from `$attrs`.

- If a component declares explicit events, corresponding `onXXX` listeners are removed from `$attrs`.

- `$attrs` essentially means **extraneous attributes,**, or "any attributes passed to the component that hasn't been explicitly handled by the component".

## Migration

This will be one of the changes that will have a bigger impact on existing code and would likely require manual migration.

- We will provide a warning when a component has unused extraneous attributes (i.e. non-empty `$attrs` that is never used during render).

- For application code that adds `class` / `style` to child components for styling purposes: the child component should be wrapped with a wrapper element.

- For higher-order components or reusable components that allow the consumer to apply arbitrary attributes / listeners to an inner element (e.g. custom form components that wrap `<input>`):

  - Declare props and events that are consumed by the HOC itself (thus removing them from `$attrs`)

  - Refactor the component and explicitly add `v-bind="$attrs"` to the target inner component or element. For render functions, apply `$attrs` with the `mergeData` helper.

  - If a component is already using `inheritAttrs: false`, the migration should be relatively straightforward.

We will need more dogfooding (migrating actual apps to 3.0) to provide more detailed migration guidance for this one, since the migration cost heavily depends on usage.
