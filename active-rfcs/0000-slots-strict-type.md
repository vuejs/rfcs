- Start Date: 2025-01-08
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/core/issues/8015, https://github.com/tobiasdiez/storybook-vue-addon/issues/74
- Implementation PR: (leave this empty)

# Summary

Allow for tools to enforce strict children type for Slots, by allowing to describe what type of children can be used in the slot.

# Basic example

```tsx
const slots = defineSlots<{
  default: () => VNode[] // any VNODE
  foo: () => HTMLInputElement // single <input/>
  foos: () => HTMLInputElement[] // multiple <input/>
  foo2: () => [HTMLInputElement, HTMLInputElement] // 2 <input/> allowed, error if children different than 2
  bar: () => SlotComponent<{ test: string }> // any Component with `test: string` prop
  baz: () => SlotComponent<{ test: string }, Comp> // Comp with `test: string` passed
}>()
```

`Tab.vue`

```vue
<script lang="ts">
import type TabItem from './TabItem.vue'
defineSlots<{
  default: () => (typeof TabItem)[]
}>()
</script>
```

`Comp.vue`

```vue
<template>
  <Tabs>
    <TabItem />
  </Tabs>
  <!--@ts-expect-error not valid children-->
  <Tabs>
    <input/>
  </Tags>
</template>
```

# Motivation

Allow greater control for library creators to specify what can be used for children.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with Vue to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

This only affects Typescript declaration, `SlotComponent` can be renamed to something else,
without a specific type to allow granualy defining the required props and the Component, it will become quite difficult to extract it in Typescript while maintainnig the correct type.

### Returns

Strict numered of children:

- Single
- Array
- Tuple

Type:

- Component
- HTMLElement
- SlotComponent

# Drawbacks

The main draw back is this is to be mostly use by tooling, vue language tools and others can get this information and provide better DX.

# Alternatives

This does not need a Vue core change, since `defineSlots` does not strict check the types, this is to be mostly adopted by tools.

# Adoption strategy

Can be adopted as soon as there's support for Vue language tools.

# Unresolved questions

- `SlotComponent` naming?
