- Start Date: 2019-04-08
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- `h` is now globally imported instead of passed to render functions as argument
- render function arguments changed and made consistent between stateful and functional components
- VNodes now have a flat data structure

# Basic example

``` js
// globally imported `createElement`
import { createElement as h } from 'vue'

export default {
  // adjusted render function arguments
  render(props, slots) {
    return h(
      'div',
      // flat data structure
      { id: props.id },
      slots.default()
    )
  }
}
```

# Motivation

In 2.x, VNodes are context-specific - which means every VNode created is bound to the component instance that created it (the "context"). This is because we need to support the following use cases (`h` is a conventional alias for `createElement`):

``` js
// looking up a component based on a string ID
h('some-component')

h('div', {
  directives: [
    {
      name: 'foo', // looking up a directive by string ID
      // ...
    }
  ]
})
```

In order to look up locally/globally registered components and directives, we need to know the context component instance that "owns" the VNode. This is why in 2.x `h` is passed in as an argument, because the `h` passed into each render function is a curried version that is pre-bound to the context instance (as is `this.$createElement`).

This has created a number of inconveniences, for example when trying to extract part of the render logic into a separate function, `h` needs to be passed along:

``` js
function renderSomething(h) {
  return h('div')
}

export default {
  render(h) {
    return renderSomething(h)
  }
}
```

When using JSX, this is especially cumbersome since `h` is used implicitly and isn't needed in user code. Our JSX plugin has to perform automatic `h` injection in order to alleviate this, but the logic is complex and fragile.

In 3.0 we have found ways to make VNodes context-free. They can now be created anywhere using the globally imported `h` function, so it only needs to be imported once in any file.

---

Another issue with 2.x's render function API is the nested VNode data structure:

``` js
h('div', {
  class: ['foo', 'bar'],
  style: { }
  attrs: { id: 'foo' },
  domProps: { innerHTML: '' },
  on: { click: foo }
})
```

This structure was inherited from Snabbdom, the original virtual dom implementation Vue 2.x was based on. The reason for this design was so that the diffing logic can be modular: an individual module (e.g. the `class` module) would only need to work on the `class` property. It is also more explicit what each binding will be processed as.

However, over time we have noticed there are a number of drawbacks of the nested structure compared to a flat structure:

- More verbose to write
- `class` and `style` special cases are somewhat inconsistent
- More memory usage (more objects allocated)
- Slower to diff (each nested object needs its own iteration loop)
- More complex / expensive to clone / merge / spread
- Needs more special rules / implicit conversions when working with JSX

In 3.x, we are moving towards a flat VNode data structure to address these problems.

# Detailed design

## Globally imported `createElement / h` function

`createElement` is now globally imported (and can be renamed to `h` as before):

``` js
import { createElement as h } from 'vue'

export default {
  render() {
    return h('div')
  }
}
```

## Render Function Arguments Change

With `h` no longer needed as an argument, the `render` function now receives a new set of arguments:

``` js
// MyComponent.js
export default {
  render(
    // declared props
    props,
    // resolved slots
    slots,
    // fallthrough attributes
    attrs,
    // the raw vnode in parent scope representing this component
    vnode
  ) {

  }
}
```

- `props` and `attrs` will be equivalent to `this.$props` and `this.$attrs` - also see [Optional Props Declaration](https://github.com/vuejs/rfcs/pull/25) and [Attribute Fallthrough](https://github.com/vuejs/rfcs/pull/26)

- `slots` will be equivalent to `this.$slots` - also see [Slots Unification](https://github.com/vuejs/rfcs/pull/20)

- `vnode` will be equivalent to `this.$vnode`, which is the raw vnode that represents this component in parent scope, i.e. the return value of `h(MyComponent, { ... })`.

Note that the render function for a functional component will now also have the same signature, which makes it consistent in both stateful and functional components:

``` js
const FunctionalComp = (props, slots, attrs, vnode) => {
  // ...
}
```

The new list of arguments should provide the ability to fully replace the current functional render context:

- `props` and `slots` have equivalent values
- `data` and `children` can be accessed directly on `vnode`
- `listeners` will be included in `attrs`
- `injections` will have a dedicated new API:

  ``` js
  import { resolveInjection } from 'vue'
  import { themeSymbol } from './ThemeProvider'

  const FunctionalComp = props => {
    const theme = resolveInjection(themeSymbol)
    return h('div', `Using theme ${theme}`)
  }
  ```

- `parent` access will be removed. This was an escape hatch for some internal use cases - in userland code, props and injections should be preferred.

## Flat VNode Data Format

``` js
// before
{
  class: ['foo', 'bar'],
  style: { color: 'red' },
  attrs: { id: 'foo' },
  domProps: { innerHTML: '' },
  on: { click: foo },
  key: 'foo'
}

// after
{
  class: ['foo', 'bar'],
  style: { color: 'red' },
  id: 'foo',
  innerHTML: '',
  onClick: foo,
  key: 'foo'
}
```

With the flat structure, the VNode data props are handled using the following rules:

- `key`, `ref` and `slots` are reserved special properties
- `class` and `style` have the same API as 2.x
- props that start with `on` are handled as `v-on` bindings, with everything after `on` being converted to all-lowercase as the event name (more on this below)
- for anything else:
  - If the key exists as a property on the DOM node, it is set as a DOM property;
  - Otherwise it is set as an attribute.

### Escape Hatches for Explicit Binding Types

With the flat VNode data structure, how each property is handled internally becomes a bit implicit. This also creates a few problems - for example, how to explicitly set a non-existent DOM property, or listen to a CAPSCase event on a custom element?

To deal with that the VNode data also supports explicit binding types via prefix:

``` js
h('div', {
  'attr:id': 'foo',
  'prop:__someCustomProperty__': { /*... */ },
  'on:SomeEvent': e => { /* ... */ }
})
```

This is equivalent to 2.x's nesting via `attrs`, `domProps` and `on`.

### Special "Reserved" Props

There are number of reserved data properties:

- `key`
- `ref`
- `slots`

In addition, you can hook into the vnode lifecycle using `vnode` prefixed hooks:

``` js
h('div', {
  vnodeMounted(vnode) {
    /* ... */
  },
  vnodeUpdated(vnode, prevVnode) {
    /* ... */
  }
})
```

These hooks are also how custom directives are built on top of.

---

Due to the flat structure, `this.$attrs` inside a component now contains any raw props that are not explicitly declared by the component, including `class`, `style`, `onXXX` listeners and `vnodeXXX` hooks. This makes it much easier to write wrapper components - simply pass `this.$attrs` down with `v-bind="$attrs"`.

## Context-free VNodes

With VNodes being context-free, we can no longer use a string ID (e.g. `h('some-component')`) to implicitly lookup globally registered components. Same for looking up directives. Instead, we need to use an imported API:

``` js
import { h, resolveComponent, resolveDirective, applyDirectives } from 'vue'

export default {
  render() {
    const comp = resolveComponent('some-global-comp')
    const fooDir = resolveDirective('foo')
    const barDir = resolveDirective('bar')

    // <some-global-comp v-foo="x" v-bar="y" />
    return applyDirectives(
      h(comp),
      this,
      [fooDir, this.x],
      [barDir, this.y]
    )
  }
}
```

This will mostly be used in compiler-generated output, since manually written render function code typically directly import the components and use them by value, and rarely have to use directives.

# Drawbacks

## Reliance on Vue Core

`h` being globally imported means any library that contains Vue components will include `import { h } from 'vue'` somewhere (this is implicitly included in render functions compiled from templates as well). This creates a bit of overhead since it requires library authors to properly configure the externalization of Vue in their build setup:

- Vue should not be bundled into the library;
- For module builds, the import should be left alone and be handled by the end user bundler;
- For UMD / browser builds, it should try the global `Vue.h` first and fallback to `require` calls.

This is common practice for React libs and possible with both webpack and Rollup. A decent number of Vue libs also already does this. We just need to provide proper documentation and tooling support.

# Alternatives

N/A

# Adoption strategy

- For template users this will not affect them at all.

- For JSX users the impact will also be minimal, but we do need to rewrite our JSX plugin.

- Users who manually write render functions using `h` will be subject to major migration cost. This should be a very small percentage of our user base, but we do need to provide a decent migration path.

  - It's possible to provide a compat plugin that patches render functions and make them expose a 2.x compatible arguments, and can be turned off in each component for a one-at-a-time migration process.

  - It's also possible to provide a codemod that auto-converts `h` calls to use the new VNode data format, since the mapping is pretty mechanical.

- Functional components using context will likely have to be manually migrated, but a similar adaptor can be provided.
