- Start Date: 2019-04-09
- Target Major Version: 3.x
- Reference Issues: vuejs/rfcs#8
- Implementation PR: N/A

# Summary

Adjust `v-model` API when used on custom components.

This builds on top of #8 (Replace `v-bind`'s `.sync` with a `v-model` argument).

# Basic example

# Motivation

Previously, `v-model="foo"` on components roughly compiles to the following:

``` js
h(Comp, {
  value: foo,
  onInput: value => {
    foo = value
  }
})
```

However, this requires the component to always use the `value` prop for binding with `v-model` when the component may want to expose the `value` prop for a different purpose.

In 2.2 we introduced the `model` component option that allows the component to customize the prop and event to use for `v-model`. However, this still only allows one `v-model` to be used on the component. In practice we are seeing some components that need to sync multiple values, and the other values have to use `v-bind.sync`. We noticed that `v-model` and `v-bind.sync` are fundamentally doing the same thing and can be combined into a single construct by allowing `v-model` to accept arguments (as proposed in #8).

# Detailed design

In 3.0, the `model` option will be removed. `v-model="foo"` (without argument) on a component compiles to the following instead:

``` js
h(Comp, {
  modelValue: foo,
  'onUpdate:modelValue': value => (foo = value)
})
```

If the component wants to support `v-model` without an argument, it should expect a prop named `modelValue`. To sync its value back to the parent, the child should emit an event named `"update:modelValue"` (see [Render Function API change](https://github.com/vuejs/rfcs/blob/render-fn-api-change/active-rfcs/0000-render-function-api-change.md) for details on the new VNode data structure).

The default compilation output prefixes the prop and event names with `model` to avoid conflict with common prop names.

RFC #8 proposes the ability for `v-model` to accept arguments. The argument can be used to denote the prop `v-model` should bind to. `v-model:value="foo"` compiles to:

``` js
h(Comp, {
  value: foo,
  'onUpdate:value': value => (foo = value)
})
```

In this case, the child component expects a `value` prop and emits `"update:value"` to sync.

Note that this enables multiple `v-model` bindings on the same component, each syncing a different prop, without the need for extra options in the component:

``` html
<InviteeForm
  v-model:name="inviteeName"
  v-model:email="inviteeEmail"
/>
```

## Handling Modifiers

In 2.x, we have hard-coded support for modifiers like `.trim` on component `v-model`. However, it would be more useful if the component can support custom modfiers. In v3, modifiers added to a component `v-model` will be provided to the component via the `modelModifiers` prop:

```html
<Comp v-model.foo.bar="text" />
```

Will compile to:

``` js
h(Comp, {
  modelValue: text,
  'onUpdate:modelValue': value => (text = value),
  modelModifiers: {
    foo: true,
    bar: true
  }
})
```

For `v-model` with arguments, the generated prop name will be `arg + "Modifiers"`:

```html
<Comp
   v-model:foo.trim="text"
   v-model:bar.number="number" />
```

Will compile to:

``` js
h(Comp, {
  foo: text,
  'onUpdate:foo': value => (text = value),
  fooModifiers: { trim: true },
  bar: number,
  'onUpdate:bar': value => (bar = value),
  barModifiers: { number: true },
})
```

## Usage on Native Elements

Another aspect of the `v-model` usage is on native elements. In 2.x, the compiler produces different code based on the element type `v-model` is used on. For example, it outputs different prop/event combinations for `<input type="text">` and `<input type="checkbox">`. However, this strategy does not handle dynamic element or input types very well:

``` html
<input :type="dynamicType" v-model="foo">
```

The compiler has no way to guess the correct prop/event combination at compile time, so it has to produce [very verbose code](https://template-explorer.vuejs.org/#%3Cinput%20%3Atype%3D%22foo%22%20v-model%3D%22bar%22%3E) to cover possible cases.

In 3.0, `v-model` on native elements produces the exact same output as when used on components. For example, `<input v-model="foo">` compiles to:

``` js
h('input', {
  modelValue: foo,
  'onUpdate:modelValue': value => {
    foo = value
  }
})
```

The module responsible for patching element props for the web platform will then dynamically determine how to actually apply them. This enables the compiler to output much less verbose code.

# Drawbacks

TODO

# Alternatives

N/A

# Adoption strategy

TODO

# Unresolved questions

## Usage on Custom Elements

Reference: [vuejs/vue#7830](https://github.com/vuejs/vue/issues/7830)

In 2.x it is difficult to use `v-model` on native custom elements, because the compiler can't tell a native custom element from a normal Vue component (`Vue.config.ignoredElements` is runtime only). The result is that given a custom element with `v-model`:

``` html
<custom-input v-model="foo"></custom-input>
```

The 2.x compiler produces [code for a Vue component](https://template-explorer.vuejs.org/#%3Ccustom-input%20v-model%3D%22foo%22%3E%3C%2Fcustom-input%3E) instead of the native default `value/input` pair.

In 3.0, the compiler will produce exactly the same code for both Vue components and native elements, and a native custom element will be handled properly as a native element.

The remaining question is that 3rd party custom elements could have unknown prop/event combinations and do not necessarily follow Vue's sync event naming conventions. For example if a custom element expects to work like a checkbox, Vue has no information on the property to bind to or the event to listen to. One possible way to deal with this is to use the `type` attribute as a hint:

``` html
<custom-input v-model="foo" type="checkbox"></custom-input>
```

This would tell Vue to bind `v-model` using the same logic for `<input type="checkbox">`, using `checked` as the prop and `change` as the event.

If the custom element doesn't behave like any existing input type, then it's probably better off to use explicit `v-bind` and `v-on` bindings.
