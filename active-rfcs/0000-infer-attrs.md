- Start Date: 2023-01-01
- Target Major Version: 3.x
- Reference Issues: [vuejs/core#3452](https://github.com/vuejs/core/issues/3452), [vuejs/core#5423](https://github.com/vuejs/core/issues/5423), [vuejs/core#6528](https://github.com/vuejs/core/discussions/6528)
- Implementation PR: [vuejs/core#7444](https://github.com/vuejs/core/pull/7444)

# Summary
Allowing to infer `attrs` by passing the second param of `defineComponent` or `defineCustomElement`. 
And in the `setup-script`,  passing generic type in the `useAttrs<T>` will also infer `attrs` to `T`.

I already published an npm package named [vue-ts-utils](https://github.com/rudy-xhd/vue-ts-utils), so that you can use `defineComponent` to infer attrs in advance.

# Basic example

## Using `defineComponent`
[TS Playground](https://www.typescriptlang.org/play?jsx=1#code/JYWwDg9gTgLgBAbzgEwKYDNgDtUGELgQ5bwC+c6UBcA5AG4CuqAtDAM7MMzAA2bNAWABQwgMZE28fODgBeFBmx4CkYjAAUCYXDhgqYNgC5E2nRQgRjAZRhRsAcwA0p0s6E6oqLGijqAlIikwq6IcACGMLZGgeFsoQBGYVDGWAwg8ahQcOSkfsJiEvDiMvIAPNJg5hCyCDQAjDTkiVA1deQA9AB8wkA) with Options Api
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

[TS Playground](https://www.typescriptlang.org/play?jsx=1#code/JYWwDg9gTgLgBAbzgEwKYDNgDtUGELgQ5bwC+c6UBcA5AG4CuqAtDAM7MMzAA2bNAWABQwgMZE28fODgBeFBmx4CkYjAAUCYXDhgqYNgC5E2nRQgRjAZRhRsAcwA0p0s6E62qGAzDq9EA0dEOABDGFs2OFIAShN3M3EsNggeVAA6Hgh7P302NPQLWIB6IrhJOyx7Ux1E5NSMrPUwiLSAIxCoYtKsBhBW1CgXYVdg5qgjRHIQyKR2qGMevoGoqOjhMQl4cRl5AB5pMHMIWQQaAEYacjmTs-IigD5hIA) with Composition Api
```tsx
const Comp = defineComponent({
  props: {
    foo: String
  },
  setup(props, { attrs }) {
    // number
    console.log(attrs.bar)
  }
}, { attrs: {} as { bar: number } })

const comp = <Comp foo={'str'} bar={1} />
```


[TS Playground](https://www.typescriptlang.org/play?jsx=1#code/JYWwDg9gTgLgBAbzgEwKYDNgDtUGELgQ5bwC+c6UBcA5AG4CuqAtDAM7MMzAA2bNAWABQwgMZE28fODgBeFBmx4CkYjAAUCYXDhgqYNgC5E2nRQgRjAZRhRsAcwA0p0s6E62qGAzDq9EA0dEOABDGFs2OFIAShN3M3EsNggeVAA6Hgh7P302NPQLWIB6IrhJOyx7Ux1E5NSMrPUwiLSAIxCoYtKsBhBW1CgXYVdg5qgjRHIQyKR2qGMevoGoqOjhMQl4cRl5AB5pMHMIWQQaAEYacjmTs-IigD5hIA) with functional Components
```tsx
import { defineComponent, h, type SetupContext } from 'vue'

type CompAttrs = {
  bar: number
  baz?: string
}

type CompEmits = {
  change: (val: string) => void;
}

const MyComp = defineComponent(
  (_props: { foo: string }, ctx: SetupContext<CompEmits, CompAttrs>) => {
    // number
    console.log(ctx.attrs.bar)
    // string | undefined
    console.log(ctx.attrs.baz)

    ctx.emit('change', '1')

    return h('div')
  }
)

const comp = <MyComp foo={'1'} bar={1} />
```


## Using `useAttrs<T>` in `script-setup`

```vue
<script setup lang="ts">
const attrs = useAttrs<{bar: number}>()
</script>
```

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

When using typescript in Vue3, the fallthrough attributes is unable to be used. It's not appropriate obviously that only one can be chosen from `typescript` and `Fallthrough Attributes`. In most cases, we choose `typescript` and set attributes to `props` option instead of using the fallthrough attributes.

Main scenes:

- Wrapping a native HTML element in a new component, such as `button`. [Here is a demo to describe](https://github.com/rudy-xhd/vue-demo/tree/native-button).
- Wrapping a component from UI library in a new component, such as `el-button` from `element-plus`. [Here is a demo to describe](https://github.com/rudy-xhd/vue-demo/tree/ui-button).

# Detailed design

## `defineComponent`
Due to typescript limitation from [microsoft/TypeScript#10571](https://github.com/microsoft/TypeScript/issues/10571), it's not possible to make generics partial in the `defineComponent` up to now. To be more clear, there is a similar question from [stackoverflow/infer-type-argument-from-function-argument-in-typescript](https://stackoverflow.com/questions/57195611/infer-type-argument-from-function-argument-in-typescript)
```tsx
// it's not work
const Comp = defineComponent<Props, Attrs>({})
```


But there still has two ways to be chosen personally.

### 1. Defining the first param that already existing, just like [vuejs/rfcs#192](https://github.com/vuejs/rfcs/pull/192) did.
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
### 2. Defining the second param as proposed.
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

At last I chosen the second way that passing `attrs` type to the second params of `defineComponent`, because I think the code of the component should not be involved just for type definition.

> [see for more detail cases](https://github.com/vuejs/core/pull/7444/files#diff-241bba82b0b4ebadd7a9c19ed82eed97283874b6d15ed32d62c05184e29ecb91R1195)

## `useAttrs<T>`
In the `setup-script`, the generic type of `useAttrs` will compile to the second param of `defineComponent`.

```vue
<script setup lang="ts">
const attrs = useAttrs<{bar: number}>()
</script>
```

Compiled Output:

```js
export default /*#__PURE__*/_defineComponent({
  setup(__props, { expose }) {
  expose();

      const attrs = useAttrs<{ bar: number }>()

return { attrs, useAttrs, ref }
}

}, { attrs: {} as { bar: number }})"
```

## `defineCustomElement`
The type inferrence of `defineCustomElement` is the same as `defineComponent`.


# Unresolved questions
Naming suggestions or improvements on the API are welcome.

