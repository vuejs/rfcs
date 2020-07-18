- Start Date: 2020-06-29
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Introduce a compile step for `<script setup>` to improve the authoring experience when using the Composition API inside Single File Components.

# Basic example

```html
<template>
  <button @click="inc">{{ count }}</button>
</template>

<script setup>
  import { ref } from 'vue'

  export const count = ref(0)
  export const inc = () => count.value++
</script>
```

# Motivation

When authoring components using the Composition API, very often `setup` is the only option that's being used. This results in some unnecessary boilerplate:

```js
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)
    const inc = () => count.value++

    return {
      count,
      inc,
    }
  },
}
```

In addition, one of the most often complained about aspect of the Composition API is the necessity to repeat all the bindings that need to be exposed to the render context using a return object.

This RFC introduces a compiler-powered alternative for the usage of `<script>` inside SFCs that greatly reduces the amount of boilerplate:

```diff
import { ref } from 'vue'

-export default {
-  setup() {
-    const count = ref(0)
+export const count = ref(0)
-    const inc = () => count.value++
+export const inc = () => count.value++

-    return {
-      count,
-      inc
-    }
-  }
-}
```

# Detailed design

When a `<script>` tag in an SFC has the `setup` attribute, it is compiled so that the code runs in the context of the `setup()` function of the component. All ES module exports are considered values to be exposed to the render context and included in the `setup()` return object.

## Using `setup()` arguments

Setup arguments can be specified as the value of the `setup` attribute:

```vue
<script setup="props, { emit }">
import { watchEffect } from 'vue'

watchEffect(() => console.log(props.msg))
emit('foo')
</script>
```

will be compiled into:

```js
import { watchEffect } from 'vue'

// setup is exported as a named export so it can be imported and tested
export function setup(props, { emit }) {
  watchEffect(() => console.log(props.msg))
  emit('foo')
  return {}
}

export default {
  setup,
}
```

## Declaring props or additional options

One problem with `<script setup>` is that it removes the ability to declare other component options, for example `props`. We can solve this by treating the default export as additional options (this also aligns with normal `<script>`):

```vue
<script setup="props">
import { computed } from 'vue'

export default {
  props: {
    msg: String,
  },
  inheritAttrs: false,
}

export const computedMsg = computed(() => props.msg + '!!!')
</script>
```

This will compile to:

```js
import { computed } from 'vue'

const __default__ = {
  props: {
    msg: String,
  },
  inheritAttrs: false,
}

export function setup(props) {
  const computedMsg = computed(() => props.msg + '!!!')

  return {
    computedMsg,
  }
}

__default__.setup = setup
export default __default__
```

Since `export default` is hoisted outside of `setup()`, it cannot reference variables declared inside. For example, if the default export object references `computedMsg`, it will result in a compile-time error.

## With TypeScript

`<script setup>` should just work with TypeScript in most cases. To type setup arguments like `props` and `emit`,, simply declare them:

```vue
<script setup="props" lang="ts">
import { computed } from 'vue'

// declare props using TypeScript syntax
// this will be auto compiled into runtime equivalent!
declare const props: {
  msg: string
}

export const computedMsg = computed(() => props.msg + '!!!')
</script>
```

The above will compile to:

```vue
<script lang="ts">
import { computed, defineComponent } from 'vue'

export default defineComponent({
  props: ({
    msg: String
  } as unknown) as undefined,
  setup(props: {
    msg: string
  }) {
    const computedMsg = computed(() => props.msg + '!!!')

    return {
      computedMsg,
    }
  }
})
</script>
```

- Runtime props declaration is automatically generated from TS typing to remove the need of double declaration and still ensure correct runtime behavior.

  - In dev mode, the compiler will try to infer corresponding runtime validation from the types. For example here `msg: String` is inferred from the `msg: string` type.

  - In prod mode, the compiler will generate the array format declaration to reduce bundle size (the props here will be compiled into `['msg']`)

  - The generated props declaration is force casted into `undefined` to ensure the user provided type is used in the emitted code.

- The emitted code is still TypeScript with valid typing, which can be further processed by other tools.


## Usage alongside normal `<script>`

There are some cases where the code must be executed in the module scope, for example:

- Declaring named exports that can be imported from the SFC file (`import { named } from './Foo.vue'`)

- Global side effects that should only execute once.

In such cases, a normal `<script>` block can be used alongside `<script setup>`:

```vue
<script>
performGlobalSideEffect()

// this can be imported as `import { named } from './*.vue'`
export const named = 1
</script>

<script setup>
import { ref } from 'vue'

export const count = ref(0)
</script>
```

the above will compile to:

```js
import { ref } from 'vue'

performGlobalSideEffect()

export const named = 1

export function setup() {
  const count = ref(0)
  return {
    count
  }
}

export default { setup }
```

## Usage Restrictions

Due to the difference in module execution semantics, code inside `<script setup>` relies on the context of an SFC. When moved into external `.js` or `.ts` files, it may lead to confusions for both developers and tools. Therefore, **`<script setup>`** cannot be used with the `src` attribute.

# Drawbacks

This is yet another way of authoring components, and it requires understanding the Composition API first.

# Adoption strategy

This is a fully backwards compatible new feature.
