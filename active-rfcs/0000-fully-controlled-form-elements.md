- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Target Major Version: 3.x
- Reference Issues: 
- Implementation PR: 

# Summary

Change `v-model` behaviour to be in full control over form elements' value.

# Basic example

Consider such a component:

```html
<template>
  <input v-model="controlledModel">
</template>

<script>
  export default {
    name: 'ControlledInput',
    data() {
      return {
        value: null,
      }
    },
    computed: {
      controlledModel: {
        get() { return this.value },
        set(value) {
          this.value = value[0] || null
        },
      }
    }    
  }
</script>
```

Currently Vue will render anything that user enters in the `<input>` even though state value is different from what the user actually sees.

After the change user only sees the first character, which perfectly corresponds to the actual state.

# Motivation

The main goal of any state-centric UI library or framework is solving the problem of UI and state synchronization. This is commonly achieved through bindings for various UI parts.

Bindings could be expressed in form of one of the following:

- **One-way binding** – UI reacts to bound state value.

  A Vue version of this is called `v-bind`. User input is handled by event listeners.

- **Two-way binding** – UI reacts to bound state value, but state value reacts to input from UI as well.

  In Vue this is called `v-model`. User input is handled by the model.

But here is the problem: what should happen when input from UI is transformed by the model or an event listener?

Moreover, what should happen when there's no event listener at all in a one-way binding case? Should any input be prevented then?

There are two ways to deal with this kind of problem: *Controlled* and *Uncontrolled* bindings.

In Controlled approach UI always corresponds to state whenever a state to UI binding is present.
In Uncontrolled approach UI is non-blocking and user input can break out of state synchronization.

A common example is an `<input>` that has state value bound to its DOM value.
An Uncontrolled version of the `<input>` would allow for any user input no matter what the actual state value is.
A Controlled version would not allow user input that doesn't correspond to final state value.

## Current state

Vue implements uncontrolled approach for form elements bindings (i.e. elements that can accept user input).

### Comparison with other frameworks

| Binding\Library | Vue          | React          | Svelte       | Angular 2+   |
|-----------------|--------------|----------------|--------------|--------------|
| Two-way binding | Uncontrolled | Not applicable | Controlled   | Uncontrolled |
| One-way binding | Uncontrolled | Controlled     | Uncontrolled | Uncontrolled |

Vue follows Angular approach to controlled bindings, while Svelte provides a nice balance between strictly controlled and uncontrolled behaviour.

[Compare Vue and Svelte model transformations behaviour](https://zkerd.sse.codesandbox.io/)

Examples of two-way binding:

```html
<input v-model="someValue">
```
```html
<!-- svelte -->
<input bind:value="someValue">
```
```html
<!-- angular -->
<input [(ngModel)]="someValue">
```

Example of one-way binding:

```html
<input :value="someValue">
```

This is considered a one-way binding as well:

```html
<input :value="someValue" @input="onInput">
```

## Problems of uncontrolled approach

Uncontrolled approach requires special handling when it comes to state synchronization.
Right now it's possible to force a controlled mode via `this.$forceUpdate()`, but this creates a couple of problems:

- Developer has to be aware of this behaviour, know about `$forceUpdate` and how it actually works
- This fix has to be applied for every uncontrolled model that has to be forced into controlled mode

Developers unaware of `$forceUpdate` fix could create their own solutions to the problem, in particular prevent native `keyup` event for validation, which is not sustainable in a long run.

And most importantly, it's an expected behaviour for `v-model`. When a two-way binding is created it is expected that the model is in a full control over the value the UI displays.

## Benefits of suggested mixed approach

Fully controlled bindings have their downsides as well:

- Clumsy uncontrolled element design (via refs in React for example). We'd like to preserve declarative style as much as possible and avoid manual state synchronization.
- Very hard to work with uncontrolled forms (i.e. when only initial value is set and no further state synchronization is needed).

  ```html
  <!-- set once and the rest is handled by the wrapping form natively -->
  <input name="foo" :value="initialValue">
  ```

  In React you'd have to create an uncontrolled input for every case like that or sync that value via event listener. Since Vue is often used to enhance an existing application this is crucial for some users.

This main focus of this RFC is to find a fair balance between controlled and uncontrolled approach, similar to what Svelte has right now. Shortly, `v-model` behaviour should change to controlled behaviour, `v-bind` should work as is.

There are many benefits that controlled `v-model` has over uncontrolled.

### Improved Developer Experience

Controlled `v-model` gives you full control over value transformations. No more need for manual state synchronization, input prevention and any other hacks that were necessary before.

### Higher-order models

There also opens up an opportunity for higher-order models to provide model-based validation or transformations.

```html
<template>
  <TextInput v-model="priceModel">
</template>

<script>
  export default {
    name: 'PriceInput',
    props: ['modelValue'],
    computed: {
      propModel: {
        get() { return this.modelValue },
        set(value) {
          // User won't be able to pass an invalid number
          if (Number.isNaN(Number(value))) return;
          this.$emit('update:modelValue', value);
        }
      }
    }
  }
</script>
```

### Uncontrolled bindings still possible

It would still be possible to have uncontrolled bindings when necessary. You can achieve that by switching to one-way bindings instead.

```html
<input :modelValue="value" @update:modelValue="onInput">
```

# Detailed design

Below is a comprehensive list of all elements affected:

* `<input>` except for `<input type="file">` which does not support `v-model`
* `<select>`
* `<textarea>`

For any of the elements on the list above the following applies:

## Two-way binding

Element's value should always correspond to `v-model` value.

For computed `v-model` that means the model getter is the source of truth for the element's value.

User authored render functions will be able to opt into the same behaviour by using `vModelDynamic` directive.

## One-way binding, no binding

Elements with one-way value bindings (`v-bind`) or no value binding at all should not be affected by this change.

## Components

Component bindings are not affected by this change since they're already controlled.

# Drawbacks

This is technically a breaking change, but it is questionable whether it does affect anyone. In order to be affected by this change developers have to be reliant on side-effects of current uncontrolled `v-model` behaviour for form elements.

# Alternatives

A `v-model.controlled` modifier has been suggested by Evan You. It can be a reasonable alternative if there's a valid case for `v-model` to lose full control over form element.
It's yet unclear how nested models should work that consist of both controlled and uncontrolled behaviour.

```html
<!-- TextInput.vue -->
<input v-model.controlled="propModel">
```
```html
<!-- is this model controlled? -->
<!-- does it work vice-versa? -->
<TextInput v-model="priceInputModel" />
```

# Adoption strategy

Docs would have to be updated to reflect these changes.
Examples showcasing differences between controlled and uncontrolled elements would be a welcome addition as well.

It's possible to provide `v-model.uncontrolled` modifier for those affected by this change to retain old behaviour.