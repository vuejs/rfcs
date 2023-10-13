- Start Date: 2023-04-07
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: https://github.com/vuejs/core/pull/8048

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

Since `<details>` and `<dialog>` are native elements, it makes sense to have built-in support for them.

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

- [Playground Before](https://play.vuejs.org/#eNp9Us1qg0AQfpXtUkgLiUJbKAQjFnrroYf2uBejo7Fxf9gdE4r47p1VY0wCPS0z8/05Y8vfjAkODfA1j1xmK4PMATYmFqqSRltkLbNQsI4VVku2IOhCKKEyrRxBd/rINh7wUKS1g0ehonCQIQEqEKSpUwSqGIu2DaJWLMnqKttvBB/pd/4VvMcw9q3LsoY1a1vf7rqeGQ7UHtI3csC0qh1bawNqlBKcJdizz9r3cACFAaa2BAw8eDKKXCNlan/jr+Gl5GPjNDepit8HIxr6aggzms/SVGmty5swWa3dLEu/orN9oa1kEnCnc8IMEtN02lb8+XHx9d7fM09RehoVUTjtmi/5cLuVTE3w47Si67Z92HHgBKcFD3KC0019LfgO0bh1GDbK7Msg0zJMaBbaRmElYZVrmTwHT8HLK9k6nPcDcHK1tfrowJKj4MuZeEjNA9iVBZWDBfuv2RX2wvBqdmPqPel/6Xj3B2p5994=)
- [Playground After](https://deploy-preview-8048--vue-sfc-playground.netlify.app/#eNp1UL1OwzAQfpXDS2GA7JUbgcTGwACjF5NcUgvbZ9lOEYry7vVPGrVDp9N335/PM3tz7uU0IdszHjqvXISAcXKtsMo48hFm8DjAAoMnA7sk3QkrbEc2JOmR/uCQBY+D1AGfhOVNjUkBCUQ0TsuICQHwnylGsvDaadX9HgRb7Q95ClY0AN80jhr3MM95vSzF2VRrkZRFj1EqHeD0bKhHvYZtITxMxkj/337VmV61Li68k7Z9ryGJzKgWrcFXTUpqGu8WDeQNGIxH6hNXxRu73dx+ftzckJuy81JabAnwZvsxtpwBN2KL3g==)

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
