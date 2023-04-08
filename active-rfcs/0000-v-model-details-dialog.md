- Start Date: 2023-04-07
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

Support built-in `v-model` binding for the native [`<details>` element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/details) and [`<dialog>` element](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/dialog).

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

  <dialog v-model="show">
    <form method="dialog">
      <button>OK</button>
    </form>
  </dialog>
</template>
```

When toggling the `<details>` or `<dialog>` element, the value of `show` should be reflected. Or once `show` has been modified, the details should expand/collapse automatically. 

# Motivation

Currently we support `v-model` built-in on `input`, `textarea`, `select`, which is convenient to bind the value to them. However, when it comes to `<details>` and `<dialog>`, `v-model` will throw and users would need to bind manually. Which could be a bit counterintuitive.

For now, users would need to manually bind them as:

```html
<script setup>
import { ref } from 'vue'

const show = ref(false)
</script>

<template>
  <details :open="show" @toggle="show = $event.target.open">
    <summary>Summary</summary>
    <span>Details</span>
  </details>

  <dialog :open="show" @close="show = false">
    <form method="dialog">
      <button>OK</button>
    </form>
  </dialog>
</template>
```

[SFC Playground](https://play.vuejs.org/#eNplUDtPwzAQ/iuHhVRYkj1KoyKxMTDA6MWkl9TCL9mXIhTlv+NHmhZ1sr677+Wb2Ytz1XlC1rA29F46goA0uY4bqZ31BDN4HGCBwVsNu0jdccNNb02I1JP9gX0iPA1CBXzmpq2LTTSIgFA7JQgjAmi/JiJr4NAr2X/vOVvlD+nlLHMAPu04KmxgntN4WbKyLtJMyYMjkpAqQGMdmtWKMzhQVl+9H/GMhioSfkSqEnkLasOktfC/3Ud5Y/N1cNk7YbrXEhSXCZUya/hNGymUHe/K9MqGmy75RNf4wXoNGulkj5FTLLbtdq3u/e3f71N+Ul6qZFkEbb3dmi1/qxufeg==)

# Detailed design

This is should be a compiler improvement, to be able to transform `v-model` on the `<details>` and `<dialog>` elements with `open` and `@toggle`.

# Drawbacks

It might theoretically conflict if users managed to implement a nodeTransformer on the user land to support `v-model` on `<details>` or `<dialog>`. But we are not aware of any existing library doing this.

# Alternatives

N/A

# Adoption strategy

This is a new feature and should not affect the existing code.

# Unresolved questions

N/A
