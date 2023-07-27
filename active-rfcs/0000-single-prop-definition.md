- Start Date: 2023-07-28
- Target Major Version: 3.x
- Reference Issues:
- Implementation PR:

# Summary

Provides a DX excellent props definition method for script setup.

# Basic example

```ts
// the prop name will be inferred from variable name
const propName = defineProp<T>()
const propName = defineProp<T>(defaultValue)
const propName = defineProp<T>(defaultValue, required)
const propName = defineProp<T>(defaultValue, required, restOptions)
const propName = defineProp<T>(defaultValue, required, restOptions, bindingName)
```

# Motivation

We align the usage of prop with ref()/computed(), which can effectively improve the fluency of thinking when writing SFC.

On the other hand, the situation where people used to dislike .value is reversed, and now most people like .value based code. For me .value is a clear hint that the value is responsive, which actually reduces the mental load.

# Detailed design

## Basic Usage

```html
<script setup>
// declare prop `count` with default value `0`
const count = defineProp(0)

// declare required prop `disabled`
const disabled = defineProp(undefined, true)

// access prop value
console.log(count.value, disabled.value)
</script>
```

## With Options

```html
<script setup>
// Declare prop with options
const count = defineProp(0, false, {
  type: Number,
  validator: (value) => value < 20,
})
</script>
```

## TypeScript

```html
<script setup lang="ts">
const count = defineProp<number>()
count.value
//    ^? type: number | undefined

// Declare prop of TS type boolean with default value
const disabled = defineProp<boolean>(true)
disabled.value
//        ^? type: boolean
</script>
```

## Reusing Props definitions

We don't prevent `defineProps` from being used together with `defineProp`, so for reuse props definitions can still use `defineProps`.

```html
<script setup>
import { sharedPropsOption } from '../shared'

const props = defineProps(sharedPropsOption)
const count = defineProp(0)
</script>
```

For consistency, users can use `toRefs` to destructure `defineProps` to refs.

```html
<script setup>
import { sharedPropsOption } from '../shared'
import { toRefs } from 'vue'

const { propA, propB } = toRefs(defineProps(sharedPropsOption))
const count = defineProp(0)

propA.value //...
</script>
```

## Props Inheritance

Since props are not collected into a single object, you need to use `$props` in the template instead of the variable define by `defineProps`.

```html
<template>
    <MyComp v-bind="$props" />
</template>
```

# Drawbacks

`defineProp` is a new API that does the same thing with `defineProps` and might cause some confusion for users in the short term.

# Alternatives

- Stay with `defineProps`
- #502
- Another API design: https://vue-macros.sxzz.moe/macros/define-prop.html#kevin-s-edition-default

# Adoption strategy

This is a new feature for backward compatibility.

# Unresolved questions

Non
