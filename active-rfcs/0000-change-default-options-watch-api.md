- Start Date: 2020-01-30
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/vue-next/issues/671
- Implementation PR:

# Summary

Vue 3 introduces the Composition API, and as part of that, a `watch` method. More information and examples can be found [here](https://vue-composition-api-rfc.netlify.com/api.html#watch).

Currently, the code for this feature is shared with the code used by the `watch` options API. However, the default behaviors are slighly different; the new `watch` method will trigger immediately after the component is mounted. This is not the case for Vue 2.X; this behavior is opt in, via the `immediate` option.

This RFC proposes to keep this change moving forward in Vue 3.x; `watch` handlers will be invoke immediately, both when using the `watch` method and the `watch` options API.

# Basic example

This is a simple component that works as-is with Vue 2.x and 3.x.

```js
const App = {
  data() {
    return {
      message: "Hello"
    };
  },
  watch: {
    message: {
      handler() {
        console.log("Immediately Triggered")
      }
    }
  }
}
```

If this component is mounted in Vue 2 the `message` handler will not be triggered until `message` is changed for the first time. A user can opt to fire the `handler` immediately by passing `immediate: true`. 

Currently in Vue 3, the `message` handle will trigger immediately; this is the default behavior in Vue 3 - the opposite of Vue 2. To opt out of this behavior in Vue 3, the user can pass a `{ lazy: true }` option.

# Motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

It is better to be consistent across the Composition API and the Options API. While they look different, the two APIs Vue provides are just two different ways of accomplishing the same thing.

# Detailed design

The `watch` options API should behave the same as the `watch` method provided by the Composition API.

# Drawbacks

The main cost of keeping this breaking change is the migration cost for existing codebases. Having `watch` trigger immediately when upgraing to Vue 3 may cause unintended behavior. The user will need to add `{ lazy: false }` to each watch API they do not want to trigger immediately.

# Alternatives

An alternative would be to implement an `immediate` option that defaults to `false` for the `watch` Options API in Vue 3.

# Adoption strategy

If we implement this proposal, how will existing Vue developers adopt it? Is
this a breaking change? Can we write a codemod? Can we provide a runtime adapter library for the original API it replaces? How will this affect other projects in the Vue ecosystem?

- The user will need to add `{ lazy: false }` to each watch API they do not want to trigger immediately in their existing codebases. 
- A codemod to assist with this migration should be possible in most cases. 
- It should be possible to write a runtime adapter to maintain the Vue 2 behavior.

# Unresolved questions

N/A
