- Start Date: 2020-07-22
- Target Major Version: 2.x and 3.x
- Reference Issues:
- Implementation PR:

# Summary

Allow to define named `slots` and `argument` in the `typescript` to provide auto-completion and type inference.

# Basic example

```ts
defineComponent({
  slots: {
    // slot name `item`
    item: { value: Number },
  },
  // ...
})
```

This would allow us to do typechecking in the `slot` binding and on when we use that `slot`.

## Type only

```ts
defineComponent({
  slots: null as {
    // slot name `item`
    item: { value: SlotType<number> }
  },
  // ...
})
```

## SFC

```vue
<template>
  <div>
    <!-- <slot name="random" /> type error no slot defined -->
    <!-- <slot name="top" v-bind="{a: 1}" /> type error no args expected -->
    <slot name="top" />
    <div v-for="(item, i) in items" :key="i">
      <slot name="item" v-bind="{ item, i }" />
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, defineAsyncComponent, h, createSlots } from 'vue'
export default defineComponent({
  slots: {
    top: null, // no value
    item: Object as () => { item: { value: number }; i: number },
  },
  setup() {
    const items = Array.from({ length: 10 }).map((_, value) => ({ value }))
    return {
      items,
    }
  },
})
</script>
```

## Render function

```ts
export default defineComponent({
  slots: {
    top: null, // no value
    item: Object as () => { item: { value: number }; i: number },
  },
  setup(_, { slots }) {
    const items = Array.from({ length: 10 }).map((_, value) => ({ value }))

    return () => {
      return h('div', [
        //slots.top({a: 1}), // type error no args expected
        //slots.random({a: 1}), // type error no slot defined
        slots.top(),
        items.map((item, i) => h('div', slots.item && slots.item({ item, i }))),
      ])
    }
  },
})
```

## Usage

When we use those component we would be able to check if the slot exists and infer the type

## SFC

```vue
<template>
  <slot-comp>
    <template v-slot:top2>Error here, `top2` not expected </template>

    <template v-slot:top>top</template>
    <template v-slot:item="{ item }">
      <p>{{ item.value }}</p>
      {{ item.random }}
      <!--  Error `random` not from type `{ value: number }`-->
    </template>
  </slot-comp>
</template>
```

# Motivation

Having type validation when using `slots`.

This will allow to have type inference with using render funcions `h` and it will allow extensions (`vetur`, etc) to be able to provide the correct types on `SFC`

# Detailed design

Implementation will be similar to `emit` typings. This can also be used at the run-time to validate the slot as we do with props.

# Drawbacks

This is optional, for large applications this will be useful.

# Alternatives

There's `web-types.json` (Jetbrains) that describes the slots. AFAIK no solution for `Vetur`

# Adoption strategy

# Unresolved questions

Should we do a pure typescript or allow to do similar prop validation?
