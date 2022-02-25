- Start Date: 2022-02-13
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/core/issues/5218
- Implementation PR:

# Summary

Introduce a macro in script setup, `defineOptions`, to use Options API in `script setup`, specifically to be able to set `name`, `inheritAttrs` and `render` in `script setup`.

# Basic example

```vue
<script setup lang="ts">
import { useSlots } from 'vue'

defineOptions({
  name: 'Foo',
  inheritAttrs: false
})

// Setup code...
const slots = useSlots()
</script>
```

<details>
<summary>Compiled Output</summary>

```js
const __default__ = {
  name: 'Foo',
  inheritAttrs: false
}
const setup = () => {
  const slots = useSlots()
  return { slots }
}
export default Object.assign(__default__, {
  setup
})
```

</details>

### JSX in `script-setup`

```vue
<script setup lang="tsx">
defineOptions({
  render() {
    return <h1>Hello World</h1>
  }
})
</script>
```

<details>
<summary>Compiled Output</summary>

With [Babel plugin](https://github.com/vuejs/babel-plugin-jsx).

```js
const __default__ = {
  render() {
    return h('h1', {}, () => 'Hello World')
  }
}
const setup = () => {}
export default Object.assign(__default__, {
  setup
})
```

</details>

# Motivation

This proposal is mainly to set the Options API using only `script setup`.

We already have `<script setup>`, but some options still need to be set in normal script tags (such as `name` and `inheritAttrs`). The goal of this proposal is to allow users to avoid script tag at all.

# Detailed design

## Overview

The behavior of `defineOptions` is basically the same as `defineProps`.

- `defineOptions` is only enabled in `<script setup>` and without normal script tag.
- `defineOptions` is **compiler macros** only usable inside `<script setup>`. They do not need to be imported, and are compiled away when `<script setup>` is processed.
- The options passed to `defineOptions` will be hoisted out of setup into module scope. Therefore, the options cannot reference local variables declared in setup scope. Doing so will result in a compile error. However, it _can_ reference imported bindings since they are in the module scope as well.
- The options passed to `defineOptions` cannot contain `props`, `emits` and `expose` options.

# Drawbacks

# Alternatives

# Adoption strategy

This feature is opt-in. Existing SFC usage is unaffected.

# Unresolved questions

## Appendix

### Enabling the Macros

I made a very prototype plugin [unplugin-vue-define-options](https://github.com/sxzz/unplugin-vue-define-options).

It supports Vite, Rollup, webpack, Vue CLI and ESBuild powered by [unplugin](https://github.com/unjs/unplugin).
