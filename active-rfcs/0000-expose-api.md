- Start Date: 2020-09-06
- Target Major Version: 3.x
- Reference Issues: #135, #210

# Summary

Provide the ability for components to control what is publicly exposed on its component instance. This proposal unifies #135 and #210 with additional details.

# Basic example

## Options API

```js
export defualt {
  expose: ['increment'],

  data() {
    return {
      count: 0
    }
  },
  methods: {
    increment() {
      this.count++
    }
  }
}
```

## Composition API

```javascript
import { ref } from 'vue'

export default {
  setup(props, { expose }) {
    const count = ref(0)

    function increment() {
      count.value++
    }

    // public
    expose({
      increment
    })

    // private
    return { increment, count }
  }
}
```

Here in the both cases, other components would only be able to access the `increment` method from this component, and nothing else.

# Motivation

In Vue, we have a few ways to retrieve the "public instance" of a component:

- Via template refs
- Via the `$parent` or `$root` properties

Up until now, the concept of the "public instance" equivalent to the `this` context inside a component. However, this creates a number of issues:

1. A component's public and internal interface isn't always the same. A component may have internal properties that it doesn't want to expose to other components, or a component may want to expose methods that are specifically meant to be used by other components.

2. A component returning render function from `setup()` encloses all its state inside the `setup()` closure, so nothing is exposed on the public instance, and there's currently no way to selectively expose properties while using this pattern.

3. `<script setup>` is also compiled into (2) so has the same issue.

The proposed APIs solve these issues by giving components the ability to explicitly declare publicly exposed properties.

# Detailed design

## Options API

A new option, `expose` is introduced. It expects an array of property keys to be exposed:

```js
// Child.vue
export defualt {
  expose: ['increment'],

  data() {
    return {
      count: 0
    }
  },
  methods: {
    increment() {
      this.count++
    }
  }
}
```

In a parent component:

```html
<Child ref="child" />
```

```js
this.$refs.child.increment() // ok
this.$refs.child.count // undefined
```

If `expose` is an empty array, then the component would be considered "closed" and no properties would be exposed.

> **Note:** `expose` option is only respected in the base component and ignored when used in mixins.

## Composition API

In Composition API, the 2nd argument of `setup` (aka the "setup context") now also provides the `expose` method:

```js
import { ref } from 'vue'

export default {
  setup(props, { expose }) {
    const count = ref(0)

    function increment() {
      count.value++
    }

    // public
    expose({
      increment
    })

    // private
    return { increment, count }
  }
}
```

The `expose` method expects an object of the actual values to be exposed.

### In `<script setup>`

The `defineExpose()` macro compiles into runtime `expose()` call.

```html
<script setup>
  const count = ref(0)

  function increment() {
    count.value++
  }

  defineExpose({
    increment
  })
</script>
```

See [relevant section](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0040-script-setup.md#exposing-components-public-interface) in the `<script setup>` RFC.

### Exposed Ref Unwrapping Behavior

Note the exposed instance unwraps refs similar to the normal public instance:

```js
// in child
const count = ref(0)
expose({
  count
})
```

```js
// in parent
this.$refs.child.count // 0
```

## Limiting Expose Control to Base Component

As raised in the discussions for a previous draft (#210), it would be very confusing if mixins and external composition functions are able to expose arbitrary properties. Therefore, the API is designed to restrict the capability to the base component only:

- The `expose` option is ignored when encountered in `mixins` or `extends` sources.
- The Composition API `expose` function is provided via the setup context instead of a global API (so that it cannot be imported in external composition functions) and can only be called once.

## Additionally Exposed Properties

An experimental implementation has been shipped in prior versions and we found in some cases there are patterns or tools (e.g. `vue-test-utils`) that relies on the presence of Vue built-in instance properties like `$el` or `$refs`.

The public instance with explicit `expose` thus still exposes these built-in instance properties.

# Drawbacks

N/A

# Alternatives

N/A

# Adoption strategy

This feature is additive and doesn't affect existing usage.

# Unresolved Questions

## Type Inference

Currently no Vue tooling has the ability to infer types of child component instances obtained from a ref.

A workaround is to explicitly export the public interface from the child component:

```html
<!-- Child.vue -->
<script lang="ts">
  export interface Api {
    // ...
  }
</script>
```

In parent:

```html
<script setup lang="ts">
  import { ref } from 'vue'
  import Child from './Child.vue'
  import type { Api } from './Child.vue'

  const childRef = ref<Api>()
</script>

<template>
  <Child ref="childRef" />
</template>
```

The above does not affect the runtime API design and therefore should not block this RFC from being merged.
