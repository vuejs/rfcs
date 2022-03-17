- Start Date: 2022-03-17
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/core/issues/3102
- Implementation PR: (leave this empty)

# Summary

Allow to infer generics when passing props on the `<template>`, `JSX` or `h`.

This will only affect typings used by typescript (IDE, TSX, etc) and don't bring any runtime cost.

There's 2 use cases:

- [script setup](https://vuejs.org/api/sfc-script-setup.html#script-setup) - This case requires heavy lifting on IDEs to enhance types.
- [defineComponent](https://vuejs.org/api/general.html#definecomponent) - This will require the users to enhance types, minimal work on the IDE might be required, eg: [Volar](https://github.com/johnsoncodehk/volar) should work out-of-box.

This RFC will target both User and IDE, distinction will be explicit

# Basic example

## Script setup - IDE

Simple `T`

```html
<script setup lang="ts" generic="T">

  defineProps<{
    list: T[],
    modelValue?: T
  }>

  defineEmits<{
    (e: `update:ModelValue`, a: T): void
  }>()
</script>
```

With `extends`

```html
<script setup lang="ts" generic="T extends string">
  defineProps<{
    types: T
  }>
</script>
```

Multiple (from https://github.com/vuejs/core/issues/3102)

```html
<script
  setup
  lang="ts"
  generic="Clearable extends boolean, ValueType extends string | number | null | undefined"
>
  type OnChange<ValueType, Clearable> = Clearable extends true
    ? (value: ValueType | null) => void
    : (value: ValueType) => void;

  defineProps<{
    clearable?: Clearable;
    value?: ValueType;
    onChange?: OnChange<ValueType, Clearable>;
  }>
</script>
```

with imported types + `extends`

> The type **can** be imported either inside of the `script setup` or in other `script`

```html
<script setup lang="ts" generic="T extends MyInterface">
  import { MyInterface } from './types'

    defineProps<{
      types: T
    }>
</script>
```

## DefineComponent - User

This is not user friendly and is more intended to be used for library creators

> Limitations: We **must** have a generic constructor to be able to infer types when using TSX,
> because of the `defineComponent` you cannot have a constructor (unless with Class API or by short the type inference), meaning the only solution we have is by appending the correct type to the `defineComponent` and let typescript use that.

Simple `T`

```ts
declare class CompProps<T> extends ComponentPropsOverride<{ list: T[], modelValue?: T}> {};

export default makeGeneric(
  defineComponent({
    props: {
      list: {
        type: Array,
        required: true
      },
      modelValue: null
    }
  })
)).withGenerics<typeof CompProps>()
```

Multiple (from https://github.com/vuejs/core/issues/3102)

```ts
type OnChange<ValueType, Clearable> = Clearable extends true
  ? (value: ValueType | null) => void
  : (value: ValueType) => void;

declare class CompProps<
  Clearable extends boolean,
  ValueType extends string | number | null | undefined> extends ComponentPropsOverride<{
    clearable?: Clearable;
    value?: ValueType;
    onChange?: OnChange<ValueType, Clearable>;
  }> {};

export default makeGeneric(
  defineComponent({
    props: {
      clearable: Boolean,
      value: [String, Number],
      onChange: Function
    }
  })
)).withGenerics<typeof CompProps>()
```

# Motivation

Provide a way to support `Generics`, since Vue3 has first class support for Typescript the only place it lacks is allowing developers to clearly express the types the component expects, instead of relying on catch all types (eg: Object, Array, null, etc).

The usage with `defineComponent` is not great but it works today (using Volar), because of the requirement of having a constructor that has a generic, there's no other way (that I can think of) to solve this in a cleaner way.

I expect the usage with `script setup` will bring this feature to be used more, since the API is simple and we have already the step of compiling it.

# Detailed design

To be able to understand this RFC we first need to understand that Types (typescript) and Implementation(javascript), might need to be handle differently, sometimes we need to patch the `defineComponent` for it to reflect a more accurate component.

## Script Setup - IDE

IDE extensions (eg: Volar), will require to do heavy lifting here, the design will be provided in TSX since Vue3 already supports first class and it should be supported **today** - if you doubt go to [typescript playground](https://www.typescriptlang.org/play?jsx=1#code/JYWwDg9gTgLgBAbzgEwKYDNgDtUGELgQ5bwC+c6UBcA5AG4CuqNAsAFDtA) and paste the results there.

### API

This will add a new attribute named `generic`, inside of that argument is a valid typescript language and ideally we should have intellisense.

`generic` value is basically the content inside `class MyClass<${genericValue}>`.
Arguments specified inside of `generic` will be available and have intellisense inside of the `script setup`.

This conversions are high-level and simplified, but still be valid on typescript only environment

Typescript to component - **NOT GENERIC COMPONENT**

```html
<script setup lang="ts" generic="T extends string">
  defineProps<{
    modelValue: T
  }>
  // code
</script>
```

Converted to:

```ts
export default (<T extends string>() => {
  return defineComponent({
    props: {
      modelValue: String as Prop<T>
    }
  })
})()
```

### Generic Components

Converting `script setup` to typescript compatible code won't make your component be able to infer the types when used on the `<template>`, you will only get statically typed components ([Generically Typed Vue.js Components](https://logaretm.com/blog/generically-typed-vue-components/))

#### Simple `T`

SFC

```html
<script setup lang="ts" generic="T extends string | number">

  defineProps<{
    list: T[],
    modelValue?: T
  }>
</script>
```

[Result playground](https://www.typescriptlang.org/play?jsx=1#code/JYWwDg9gTgLgBAbzgEwKYDNgDtUGELgQ5YwA0cAClBGACoCeYqcAvnOtSHAOQBuArqm4BYAFBiAxkQDO8fODgBeOAAoAPLTioAHjFRZk0uLKjYA5nAA+cLPxAAjVFAB8KgJRLniMXDhRUMPxQWCgY2HgEkMQwKgg+vnBg1GDSAFzeogkJADbAsulxmVkJMIyo6QCCUFAAhvRwNUZUNAxMGgDaALrOpPHFfqgAjvzA-sjpMFCCfQksvUVZIBBo2QBqNdmC6bbZ2Q1G-FgA1lgQAO4hjZTJragazjMs8SxuYi-uYmJoEtk1-nA-RpGeRgADi+icwAkGi0un0hmMk3MVhsdkcLgyvn8NWQRGy9QAJEkaGlMTk8jB0rQujMlit1ptUAB+KnPN5iHSQWChdA1fjZOSRfZwUpMCDoOAguAAMhFZXFksi4JwpgkcAA9Oq4BBeE5TGgRRA4GYIaq4AADInJaTmz6iTUi1CyL6oQH-KRYWRwACy9BBE3lEqlstFqAVIOVkIkYgA3GpfVLcrJFAh2gBGTpsdUPURxhNCpMwFPpzNwOmoNYbQQptNZnMOgACMGkAFpOa6YG3qtBY-G-QWKcWM2xy5XGSnuGnuHWgA)

```ts
import { defineComponent, PropType } from 'vue'

const Comp = (<T extends string | number>() => {
  return defineComponent({
    props: {
      list: {
        type: Array as PropType<T[]>,
        required: true
      },
      modelValue: null as unknown as PropType<T>
    }
  })
})()

declare class CompGeneric<T extends string | number> {
  readonly $props: {
    list: T[]
    modelValue?: T
  }
}
export default Comp as typeof Comp & typeof CompGeneric // override to generic `$props`

// test
declare const MyComp: typeof Comp & typeof CompGeneric
;<MyComp list={[1]} />
;<MyComp list={[1]} modelValue={1} />
// @ts-expect-error
;<MyComp list={[1]} modelValue={'1'} />
```

### extends type

SFC

```html
<script setup lang="ts" generic="T extends MyItem">
  interface MyItem {
    name: string;
    foo: number
  }

  defineProps<{
    list: T[],
    modelValue?: T
  }>
</script>
```

[Result playground](https://www.typescriptlang.org/play?jsx=1#code/JYWwDg9gTgLgBAbzgEwKYDNgDtUGELgQ5YwA0cAClBGACoCeYqcAvnOtSHAOQBuArqm4BYAFBjsMVFHQBDAMbMAsvQCSUrgjFwdcLLJCoAXHADOMKNgDmAbm270ECCaz8QAI2liWYsfKLmcPjgcAC8cAAUADy0cKgAHlJYyKZwKuqoIAB8EQCUYVmI9jpQqDD8UFgoGNh4BJDEMBFaorptcGDUYKYmLe39cAA2wOa9xQP9MIzGcACCUFCy9HCyqVQ0DEwxANoAulmk4xNtpQCO-MClyCYWgkcDLIetxzogEGiDAGqyg4Iu-INBitUvwsABrLAQADuVVWlC6m1QMSy9x0PmeaNy3lyeV8ojQ8kGslKcEJq1SwTAAHFUDhLPIYnFErSUmk1BpCn1dKVZMgiINlgASTo0HpFDHtYajOC0PaouBvD7fX6oAD8Jlo43R6ISkFg1TkAPglOBcCmTAg6CC9TgADIzdNLdbwDS6cB5HAAPSeuAQXjSSxoM0QOBWWnSd1wAAGwq6pijeO9ZtQ5jEBKJJP8WECKkpN0dVpN9vNqCdlNdEfkYhscCiuZtUpgoQQ2yQ+kMJm4UnM3HIjmccAAjKxdmxPSjRDW6-QTY3m629AYZl2UzBe+wnCZhyxRwr3qgvj9BM220vO92133N0PWGOJ0mAAIwUwAWl1qHkMDfC2g1dr9ZCOcW0HXdFQPZVjwQbhB24O8xEfZ833iJhP2-agoD-adZxGJsW1PDseAvdd+y3Ec2DAw8VWbQc7yAA)

```ts
import { defineComponent, PropType } from 'vue'

interface MyItem {
  name: string
  foo: number
}

const Comp = (<T extends MyItem>() => {
  return defineComponent({
    props: {
      list: {
        type: Array as PropType<T[]>,
        required: true
      },
      modelValue: null as unknown as PropType<T>
    }
  })
})()

declare class CompGeneric<T extends MyItem> {
  readonly $props: {
    list: T[]
    modelValue?: T
  }
}
export default Comp as typeof Comp & typeof CompGeneric // override to generic `$props`

// test
declare const MyComp: typeof Comp & typeof CompGeneric
;<MyComp list={[{ name: 'test', foo: 1 }]} />
;<MyComp
  list={[{ name: 'test', foo: 1 }]}
  modelValue={{ name: 'test', foo: 1 }}
/>
// @ts-expect-error
;<MyComp list={[1]} modelValue={'1'} />
// @ts-expect-error
;<MyComp list={[{ name: 'test', foo: 1 }]} modelValue={1} />
```

## DefineComponent

Is not required any further work on IDE, the only work needed is to expose this types on Vue3 package:

[Playground](https://www.typescriptlang.org/play?jsx=1#code/JYWwDg9gTgLgBAbzgEwKYDNgDtUGELgQ5YwA0cAIhtngZMfAL5zpQFwDkAbgK6ocBYAFDC0AYwA2AQyioWPLGJjAicVAA8wqJQBUAnloA8OgHwAKKQC44OgJTWuEYMgDcw4QHoPcMGy7PUZDgAIz04XlRhDUhYFG1pWTgpYIBnGCgpJThJKRSUuHxCBgAFNjAUgHkuVCgoAMNSiHK4AF5ERhNEYTgeuFkpZCIJMIASXyaU60by4UZ3IXEEuXQFJRUsOBApAGtUAHFUHDqxY3MxaztrJAB3YBgACwOj4DEUhrL8jRhD5HykcZgEBgBlQ1kK9EOMGmlWqtXqUiwek6HTM9hscAAZHBoXA5kIvHBvON-GggqFwnx5gSeCkavNgVo4BUsLh7giAOaoQwANSkEj4+i05FwElQMmSos6bRFYoywVFanU3ywvzg6UpQl6cAA-HAzFw+XxrLz+ahBXIAD5wLA8CQSWytTqOZzdXrWfWG0FwE0CkEOlpOpzIebYb5QdCZORPGovaaGGXi+Wocg+s0gzoIV09SSyiWobVg0WJ0VZ8Kegvez3m0tEVkc-PWZl1rCcnlVkHCotyyWzeaLGRyHJ5Ap0aGGUsJ7tyL4-fLBCAQItYUil1PmxXK1VpOotuBWm0gYI1PfW20SE8KNCYHDBzW9TozlX5cFESHQqo1OpoQzR45xyd5im7ZaCYGa9iIEHCGIRBpHAACyei-i84KtJsOz7IcMZiGYV40C+DBmJmd49OM5RXKWWrfGk1gAEILkupZ4owti2AAdLcDxIa8hgMqgEDoCO4DQuYthuBBLhwIYCFceCpYGqaLQIBweQpBweJajmxaRMRcC1myLaoIpUiOl0OlatE2gwOahjKXkHAnjadrmFItiMXiHgmEAA)

```ts
// not required, just a sugar for users
export declare abstract class ComponentPropsOverride<Props = {}> {
  readonly $props: Props
}

export declare function makeGeneric<T>(c: T): {
  withGenerics<Props extends { prototype: ComponentPropsOverride<any> }>(): T &
    Props
}
```

```ts
import {
  defineComponent,
  DefineComponent,
  ComponentPropsOverride,
  makeGeneric
} from 'vue'

// user

type OnChange<ValueType, Clearable> = Clearable extends true
  ? (value: ValueType | null) => void
  : (value: ValueType) => void

interface GenericProp<Clearable, ValueType> {
  clearable?: Clearable
  value?: ValueType
  onChange?: OnChange<ValueType, Clearable>
}

declare class CompProps<
  Clearable extends boolean,
  ValueType extends string | number | null | undefined
> extends ComponentPropsOverride<GenericProp<Clearable, ValueType>> {}

const MyGenericComp = makeGeneric(
  defineComponent({
    props: {
      test: Boolean
    }
  })
).withGenerics<typeof CompProps>()

;<MyGenericComp
  value={'sss'}
  clearable
  onChange={(a) => {
    expectType<'sss' | null>(a)
  }}
/>
```

# Drawbacks

## Script Setup

This is the way with less drawbacks, DX should be great and it will be the preferred way.

## defineComponent

This brings quite a lot of work and DX is not amazing, this will require the user to be quite strict to prevent wrong type inferences, because props will be duplicated, errors are easy to show up.

# Alternatives

`defineComponent` changes can be already be done on the `userLand`, but it would be good to provide guidance on how to do this, I expect libraries to take advantage of this to provide better type for their components

# Adoption strategy

This functionality is additive, we should only update the components that are actually generic.

# Unresolved questions

Naming suggestions or improvements on the API are welcome.
