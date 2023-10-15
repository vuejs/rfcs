- Start Date: 2020-03-05
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

Add a `prop` built-in modifier to `v-model` in order to support two-way binding for props using events.

# Basic example

With a `prop` modifier you'd able to create transparent `v-model` wrappers without extra hassle:

```html
<Foo v-model.prop="value" />
```

Is equivalent to:

```html
<Foo :value="value" @input="$emit('input', $event)" />
```

---

Named models are also supported:

```html
<Foo v-model:bar.prop="bar" />
```

Which is equivalent to:

```html
<Foo :bar="bar" @update:bar="$emit('update:bar', $event)" />
```


# Motivation

Creating wrapper components around anything that uses `v-model` could be considered a somewhat cumbersome process at the moment.

## Built-in models

Firstly, you'll have to know what you're dealing with when using `v-model`: an html element or a component.
It's important to distinguish between those because built-in inputs for example already have a custom behaviour for `v-model`.
In order to preserve it you'll have to use a computed `v-model`:

```js
export default {
  computed: {
    model: {
      get() { return this.value }
      set(value) { this.$emit('input', value) }
    }
  }
}
```

Simply using `@input="$emit('value', $event)"` on `<input>` for example won't work because you'll be working with a DOM event, that's not processed by Vue's built-in model.
You'll have to do this for each html element that has built-in `v-model` processing done by Vue (these include: `<input type="text">`, `<input type="checkbox">`, `<input type="radio">`, `<textarea>` and `<select>`).

With a `prop` modifier there's no such an issue anymore. You simply write `v-model.prop="value"` and the binding is created automatically using events. There's no longer any need to distinguish between html elements and components.

Secondly, you'll be free of creating model wrappers manually for components, so this code is no longer needed:

```html
<Foo :value="value" @input="$emit('input', $event)" />
```

And can be replaced with a single `v-model`:

```html
<Foo v-model.prop="value" />
```

## Argument models

With the introduction of arguments in `v-model` there's also a problem with wrapping components that expect you to use `v-model` with arguments.
Consider this example:

```html
<Foo v-model:bar="bar" />
```

In case we're wrapping such a component using props the code will look as following:

```html
<Foo :bar="bar" @update:bar="$emit('update:bar', $event)" />
```

This is not very user-friendly compared to simply `v-model:bar="bar"` when dealing with local state.
Wrapper components should not introduce extra barriers when working with `v-model`.

## Why modifier

One may consider having simply `v-model` to deal with both local state binding and event binding.
Modifier is required in order to solve two problems with this approach:

1. Performance penalty of checking against props
2. Readability (not being able to distinguish between data that's local and external)

# Detailed design

`prop` modifier changes `v-model` behaviour.
When applied, `v-model` should no longer mutate model value, but instead emit an event that corresponds to model name with a new model value.

Provided we have such a template:

```html
<Foo v-model:bar.prop="bar" />
```

Would compile to this render function:

```js
export default {
  render() {
    h('Foo', {
      bar: this.bar,
      'onUpdate:bar': (value) => { this.$emit('update:bar', value) },
      barModifiers: {
        prop: true
      }
    })
  }
}
```

`prop` modifier should also gracefully handle interaction with built-in `v-model` directives such as `vModelSelect`, `vModelText`, `vModelCheckbox` and `vModelRadio` that are done on a compiler level.

`prop` modifier is also conflict-free for any other model modifiers, except for `prop` modifier itself.

## Computed models

`prop` modifier does not support model value as a computed property with getter and setter. An error should be raised in that case.

```html
<template>
  <input v-model.prop="computedModel">
</template>

<script>
  export default {
    computed: {
      computedModel: {
        // should raise an error on component render
        get() {},
        set() {},
      }
    }
  }
</script>
```

## Deep models

Deep models should return a cloned value with a model value property replaced.

```html
<template>
  <input v-model.prop="modelValue.title">
</template>

<script>
  export default {
    props: ['modelValue'],
  }
</script>
```

In the example above `v-model` setter should do the following:

```js
set(titleValue) {
  const newValue = Object.assign({}, this.modelValue);
  newValue.title = titleValue;
  this.$emit('update:modelValue', newValue);
}
```

For arrays it should be:

```html
<template>
  <input v-model.prop="modelValue[1]">
</template>

<script>
  export default {
    props: ['modelValue'],
  }
</script>
```
```js
set(titleValue) {
  const newValue = this.modelValue.slice(0);
  newValue[0] = titleValue;
  this.$emit('update:modelValue', newValue);
}
```

# Drawbacks

A new reserved modifier for model.
There's already a `prop` modifier for `v-bind` which serves a completely different task.
Having two modifiers with a same name could result in confusion.

# Alternatives

An alternative `event` modifier name could be considered to better indicate a type of binding we're using.

# Adoption strategy

No migration steps required.
