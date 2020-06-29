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
      inc
    }
  }
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

The following equivalent of `setup()` arguments are implicitly available inside `<script setup>`:

- `$props`
- `$attrs`
- `$slots`
- `$emit`

The example code

```js
import { watchEffect } from 'vue'

watchEffect(() => console.log($props.msg))
$emit('foo')
```

would be compiled into:

```js
import { watchEffect } from 'vue'

export default {
  setup($props, { emit: $emit }) {
    watchEffect(() => console.log($props.msg))
    $emit('foo')
    return {}
  }
}
```

## Declaring props or additional options

One problem with `<script setup>` is that it removes the ability to declare other component options, for example `props`. We can solve this by checking a special export, `$options`:

```vue
<script setup>
import { computed } from 'vue'

export const $options = {
  props: {
    msg: String
  }
}

export const computedMsg = computed(() => $props.msg + '!!!')
</script>
```

This will compile to:

```js
import { computed } from 'vue'

const $options = {
  props: {
    msg: String
  },
  inheritAttrs: false
}

export default {
  ...$options,
  setup($props) {
    const computedMsg = computed(() => $props.msg + '!!!')

    return {
      computedMsg
    }
  }
}
```

## With TypeScript

`<script setup>` should just work with TypeScript in most cases. To make implicitly injected variables like `$props` and `$emit` work with proper types, simply declare them:

```vue
<script setup lang="ts">
import { computed } from 'vue'

// declare props using TypeScript syntax
// this will be auto compiled into runtime equivalent!
declare const $props: {
  msg: string
}

export const computedMsg = computed(() => $props.msg + '!!!')
</script>
```

The above will compile to:

```vue
<script lang="ts">
import { computed, defineComponent } from 'vue'

type __$props__ = { msg: string }

export default defineComponent({
  props: (['msg'] as unknown) as undefined,
  setup($props: props__) {
    const computedMsg = computed(() => $props.msg + '!!!')

    return {
      computedMsg
    }
  }
})
</script>
```

- Runtime props declaration is automatically generated from TS typing to remove the need of double declaration and still ensure correct runtime behavior. (It is force casted into `undefined` to ensure the user provided type is used in the emitted code)

- The emitted code is still TypeScript with valid typing, which can be further processed by other tools.

# Drawbacks

This is yet another way of authoring components, and it requires understanding the Composition API first.

# Adoption strategy

This is a fully backwards compatible new feature.


# Unresolved Questions

## Magic `let` bindings

It is technically possible to compile `let` bindings in a way so that root level `let` bindings are implicitly reactive:

```vue
<script setup>
export let count = 0

export function increment() {
  count++
}
</script>
```

can be compiled into:

```vue
<script>
import { reactive } from 'vue'

export default {
  setup() {
    const __ctx__ = shallowReactive({})
    __ctx__.count = 0

    function increment() {
      __ctx__.count++
    }

    return Object.assign(__ctx__, {
      increment
    })
  }
}
</script>
```

This makes the code even more succinct and removes the need to use `ref` in the root scope of the component. However, this may be a bit too magical and there are a number of consistency issues if we enable this behavior:

- Objects are not implicitly reactive:

  ```js
  export const state = { count: 0 }

  export function increment() {
    // doesn't work
    state.count++
  }
  ```

  Making root scope objects deeply reactive by default can lead to potential performance problems, since the user may declare a 3rd party object of unknown size.

- Doesn't work inside nested functions:

  ```js
  function useFeature() {
    // not reactive, since it's not root level
    let count = 1
    const inc = () => count++

    return {
      count,
      inc
    }
  }
  ```

  If we also compile functions inside `<script setup>`, then composition functions inside and outside SFCs will behave *very* differently and the mental model is no longer consistent. It also creates friction for extracting reusable functions out of components.

- Dependency tracking no longer explicit:

  ```js
  export let count = 0

  watchEffect(() => console.log(count))
  ```

  By looking at the `watchEffect` callback alone, it is not immediately clear whether it tracks any dependencies at all. We will now need to check if a variable is declared via root level `let` to be sure.

Given the above considerations, magical `let` bindings is not a part of the proposed features of this RFC, but it's discussed here to provide context on why we chose not to include it.
