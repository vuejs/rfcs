- Start Date: 2019-06-17
- Target Major Version: 2.x / 3.x
- Reference Issues:
- Implementation PR:


# Summary

Make component props required out of the box. Infer optional props from the already-existing `default` key.

# Basic example

```js
const Component = {
  props: ['title'], // required prop
}
```

```js
const Component = {
  props: {
    // required prop
    title: String,

    // required prop
    subTitle: {
      type: String
    },

    // optional prop
    description: {
      type: String,
      default: 'My default value'  
    }
  }
}
```


# Motivation

There are two main motivators behind this proposal.

Firstly, current prop declaration API allows users to set illogical scenarios, such as providing a default value for a required prop. There's even a specific [`Vue Eslint rule`](https://eslint.vuejs.org/rules/require-default-prop.html#vue-require-default-prop) to warn developers about it.

Secondly, this proposal lets users write a shorter, terser syntax to define props by ditching the object notation and relying solely on the `name: Type` syntax while defining required props. From my experience both using and teaching Vue, setting a prop as required is the most common scenario, so this would reduce some boilerplate on Vue components.


# Drawbacks

People familiar with current implementation would need to unlearn current behavior.

The `default` key would become somewhat "overloaded", as it would not only provide a default value, but also mark the prop as optional. I see this as a feature.


# Alternatives

A possible alternative is to keep using the `required` attribute and let user use it to explicitly express the intended behavior:

```js
const Component = {
  props: {
    // required prop
    title: String,

    // optional prop
    subTitle: {
      type: String,
      required: false,
    },

    // optional prop
    description: {
      type: String,
      required: false,
      default: 'My default value'
    }
  }
}
```


# Adoption strategy

A codemod could remove all unnecessary `required: true` statements.

After that, and depending on the prop declaration syntax used on each codebase, developers would need to fix missing required prop warnings.


# Unresolved questions

> Is this a breaking change?

While this proposal changes the public API (and thus it could be considered a breaking change), existing applications shouldn't break - they'd only throw additional console warnings.