- Start Date: 2019-02-27
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR:

# Summary

Make it explicit what events are emitted by the component.

# Basic example

```javascript
const Comp = {
  emits: {
    submit: payload => {
      // validate payload by returning a boolean
    }
  },

  created() {
    this.$emit('submit', { /* payload */ })
  }
}
```

# Motivation

- **Documentation:** Similar to `props`, explicit `emits` declaration serves as self-documenting code. This can be useful for other developers to instantly understand what events the component is supposed to emit.

- **Runtime Validation:** The option also offers a way to perform runtime validation of emitted event payloads.

- **Type Inference:** The `emits` option can be used to provide type inference so that `this.$emit` and `setupContext.emit` calls can be typed.

- **IDE Support:** IDEs can leverage the `emits` option to provide auto-completion when using `v-on` listeners on a component.

- **Listener Fallthrough Control:** With the proposed attribute fallthrough changes, `v-on` listeners on components will fallthrough as native listeners by default. `emits` provides a way to declare events as component-only to avoid unnecessary registration of native listeners.

# Detailed design

A new optional component option named `emits` is introduced.

## Array Syntax

For simple use cases, the option value can be an Array containing string events names:

```javascript
{
  emits: [
    'eventA',
    'eventB'
  }
}
```

## Object Syntax

Or it can be an object with event names as its keys. The value of each property can either be `null` or a validator function. The validation function will receive the additional arguments passed to the `$emit` call. For example, if `this.$emit('foo', 1, 2)` is called, the corresponding validator for `foo` will receive the arguments `1, 2`. The validator function should return a boolean to indicate whether the event arguments are valid.

```javascript
{
  emits: {
    // no validation
    click: null,

    // with validation
    //
    submit: payload => {
      if (payload.email && payload.password) {
        return true
      } else {
        console.warn(`Invalid submit event payload!`)
        return false
      }
    }
  }
}
```
## Fallthrough Control

The new [Attribute Fallthrough Behavior](https://github.com/vuejs/rfcs/blob/amend-optional-props/active-rfcs/0000-attr-fallthrough.md) proposed in [#154](https://github.com/vuejs/rfcs/pull/154) now applies automatic fallthrough for `v-on` listeners used on a component:

``` html
<Foo @click="onClick" />
```

We are not making `emits` required for `click` to be trigger-able by component emitted events for backwards compatibility. Therefore in the above exampe, without the `emits` option, the listener can be triggered by both a native click event on `Foo`'s root element, or a custom `click` event emitted by `Foo`.

If, however, `click` is declared as a custom event by using the `emits` option, it will then only be triggered by custom events and will no longer fallthrough as a native listener.

Event listeners declared by `emits` are also excluded from `this.$attrs` of the component.

## Type Inference

The Object validator syntax was picked with TypeScript type inference in mind. The validator type signature can be used to type `$emit` calls:

``` ts
const Foo = defineComponent({
  emits: {
    submit: (payload: { email: string, password: string }) => {
      // perform runtime validation
    }
  },

  methods: {
    onSubmit() {
      this.$emit('submit', {
        email: 'foo@bar.com',
        password: 123 // Type error!
      })

      this.$emit('non-declared-event') // Type error!
    }
  }
})
```

# Adoption strategy

The introduction of the `emits` option should not break any existing usage of `$emit`.

However, with the fallthrough behavior change, it would be ideal to always declare emitted events. We can:

1. Provide a codemod that automatically scans all instances of `$emit` calls in a component and generate the `emits` option.

2. Emit a runtime warning when an emitted event isn't explicitly declared using the option.
