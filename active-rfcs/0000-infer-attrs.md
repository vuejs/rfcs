- Start Date: 2023-01-01
- Target Major Version: 3.x
- Reference Issues: [vuejs/core#3452](https://github.com/vuejs/core/issues/3452), [vuejs/core#5423](https://github.com/vuejs/core/issues/5423), 
- Implementation PR: [vuejs/core#7444](https://github.com/vuejs/core/pull/7444)

# Summary
Allow to infer `attrs` when passing type on the second param of `defineComponent` and `defineCustomElement`. 
And in the `setup-script`,  passing generic type on the `useAttrs<T>` will infer `attrs` to `T`.

I already published an npm package named [vue-ts-utils](https://github.com/rudy-xhd/vue-ts-utils), so that you can use `defineComponent` to infer attrs in advance.

# Basic example

## Using `defineComponent`
[TS Playground](https://www.typescriptlang.org/play?jsx=1#code/JYWwDg9gTgLgBAbzgEwKYDNgDtUGELgQ5bwC+c6UBcA5AG4CuqAtDAM7MMzAA2bNAWABQwgMZE28fODgBeFBmx4CkYjAAUCYXDhgqYNgC5E2nRQgRjAZRhRsAcwA0p0s6E6oqLGijqAlIikwq6IcACGMLZGgeFsoQBGYVDGWAwg8ahQcOSkfsJiEvDiMvIAPNJg5hCyCDQAjDTkiVA1deQA9AB8wkA)
```tsx
const Comp = defineComponent({
  props: {
    foo: String
  },
  render() {
    // number
    console.log(this.$attrs.bar)
  }
}, { attrs: {} as { bar: number } })

const comp = <Comp foo={'str'} bar={1} />
```

## Using `useAttrs<T>` in `script-setup`

```vue
<script setup lang="ts">
const attrs = useAttrs<{bar: number}>()
</script>
```

<details>
<summary>Compiled Output</summary>

```js
export default /*#__PURE__*/_defineComponent({
  setup(__props, { expose }) {
  expose();

      const attrs = useAttrs<{ foo: number }>()

return { attrs, useAttrs, ref }
}

}, { attrs: {} as { foo: number }})"
```

</details>


## Using `defineCustomElement`
```tsx
const Comp = defineCustomElement({
  props: {
    foo: String
  },
  render() {
    // number
    console.log(this.$attrs.bar)
  }
}, { attrs: {} as { bar: number } })

```

# Motivation
This proposal is mainly to infer `attrs` using `defineComponent`.

When using typescript in Vue3, the fallthrough attributes is unable to be used. It's not appropriate obviously that only one can be chosen from `typescript` and `the fallthrough attributes`. In most cases, we choose `typescript` and set attributes to `props` options instead of using the fallthrough attributes.


# Detailed design

## `defineComponent`
Due to typescript limitation from [microsoft/TypeScript#10571](https://github.com/microsoft/TypeScript/issues/10571), it's not possible to skip generics up to now in the `defineComponent` like below.
```tsx
// it's not work
const Comp = defineComponent<Props, Attrs>({})
```

There still has two ways to be chosen.

1. Defining the first param that already existing, just like [vuejs/rfcs#192](https://github.com/vuejs/rfcs/pull/192) did.
```tsx
const Comp = defineComponent({
  attrs: {} as { bar: number },
  props: {
    foo: String
  },
  render() {
    // number
    console.log(this.$attrs.bar)
  }
})

const comp = <Comp foo={'str'} bar={1} />
```
2. Defining the second param as proposed.
```tsx
const Comp = defineComponent({
  props: {
    foo: String
  },
  render() {
    // number
    console.log(this.$attrs.bar)
  }
}, { attrs: {} as { bar: number } })

const comp = <Comp foo={'str'} bar={1} />
```

At last i chosen the second way that pass `attrs` type to the second params of `defineComponent`, because I think the code of the component should not be involved just for type definition.


The following below is the design details.
- `attrs` is inferred to `{ class: unknown; style: unknown }` when the value of the second param is `undefined`
- `attrs` is lower priority  than `props`.
- [see for more detail cases](https://github.com/vuejs/core/pull/7444/files)

## `useAttrs<T>`
In the `setup-script`, the generic type of `useAttrs` will compile to the second param of `defineComponent`.
```ts
export default /*#__PURE__*/_defineComponent({
  setup(__props, { expose }) {
  expose();

      const attrs = useAttrs<{ foo: number }>()

return { attrs, useAttrs, ref }
}

}, { attrs: {} as { foo: number }})"
```

## `defineCustomElement`
The type inferrence of `defineCustomElement` is the same as `defineComponent`.


# Unresolved questions
Naming suggestions or improvements on the API are welcome.

