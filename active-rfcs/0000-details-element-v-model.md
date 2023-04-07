- Start Date: 2023-04-07
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

Support built-in `v-model` binding for the native [`<details>` element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/details).

# Basic example

```html
<script setup>
import { ref } from 'vue'

const show = ref(false)
</script>

<template>
  <details v-model="show">
    <summary>Summary</summary>
    <span>Details</span>
  </details>
</template>
```

When toggling the `<details>` element, the value of `show` should be reflected. Or once `show` has been modified, the details should expand/collapse automatically. 

# Motivation

Currently we support `v-model` built-in on `input`, `textarea`, `select`, which is convenient to bind the value to them. However, when it comes to `<details>`, `v-model` will throw and users would need to bind maually. Which could be a bit counterintuitive. 

```html
<script setup>
import { ref } from 'vue'

const show = ref(false)
</script>

<template>
  <details :open="show" @toggle="show = e.target.open">
    <summary>Summary</summary>
    <span>Details</span>
  </details>
</template>
```

# Detailed design

This is should be a compiler improvement, to be able to transform `v-model` on the `<details>` element with `open` and `@toggle`.

# Drawbacks

It might threoctially conflict if users managed to implement a nodeTransformer on the user land to support `v-model` on `<details>`. But we are not aware of any existing library doing this.

# Alternatives

N/A

# Adoption strategy

This is a new feature and should not affect the existing code.

# Unresolved questions

N/A
