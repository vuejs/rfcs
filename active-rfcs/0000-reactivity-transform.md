- Start Date: 2021-11-26
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/rfcs/pull/182, https://github.com/vuejs/rfcs/pull/228, https://github.com/vuejs/rfcs/pull/368, https://github.com/vuejs/rfcs/discussions/413

# Summary

Introduce a set of compiler transforms to improve ergonomics when using Vue's reactivity APIs, specifically to be able to use refs without `.value`.

# Basic example

```ts
// declaring a reactive variable backed by an underlying ref
let count = $ref(1)

// log count, and automatically log again whenever it changes.
// no need for .value anymore!
watchEffect(() => console.log(count))

function inc() {
  // assignments are reactive
  count++
}
```

<details>
<summary>Compiled Output</summary>

```js
import { watchEffect, ref } from 'vue'

const count = ref(1)

watchEffect(() => console.log(count.value))

function inc() {
  count.value++
}
```

</details>

# Motivation

Ever since the introduction of the Composition API, one of the primary unresolved questions is the use of refs vs. reactive objects. It can be cumbersome to use `.value` everywhere, and it is easy to miss if not using a type system. Some users specifically lean towards using `reactive()` exclusively so that they don't have to deal with refs.

This proposal aims to improve the ergonomics of refs with a set of compile-time macros.

# Detailed design

## Overview

- Every ref-creating API has a `$`-prefixed macro version, e.g. `$ref()`, that creates **reactive variables** instead of refs.
- Destructure objects or convert existing refs to reactive variables with `$()`
- Destructuring `defineProps()` in `<script setup>` also creates reactive variables.
- Retrieve the underlying refs from reactive variables with `$$()`

## Reactive Variables

To understand the issue we are trying solve, let's start with an example. Here's a normal ref, created by the original `ref()` API:

```js
import { ref, watchEffect } from 'vue'

const count = ref(0)

watchEffect(() => {
  // access value (track)
  console.log(count.value)
})

// mutate value (trigger)
count.value = 1
```

The above code works without any compilation, but is constrained by how JavaScript works: we need to use the `.value` property, so that Vue can intercept its get/set operations in order to perform dependency tracking and effect triggering.

Now let's look at a version using **reactive variables**:

```js
import { watchEffect } from 'vue'

let count = $ref(0)

watchEffect(() => {
  // access value (track)
  // compiles to `console.log(count.value)`
  console.log(count)
})

// mutate value (trigger)
// compiles to `count.value = 1`
count = 1
```

This version behaves exactly the same as the original version, but notice that we no longer need to use `.value`. In fact, this makes our JS/TS code work the same way as in Vue templates where root-level refs are automatically unwrapped.

The `$ref()` function is a **compile-time macro** that creates a **reactive variable**. It serves as a hint to the compiler: whenever the compiler encounters the `count` variable, it will automatically append `.value` for us! Under the hood, the reactive variable version is compiled into the normal ref version.

Every reactivity API that returns refs will have a `$`-prefixed macro equivalent. These APIs include:

- `ref` -> `$ref`
- `computed` -> `$computed`
- `shallowRef` -> `$shallowRef`
- `customRef` -> `$customRef`
- `toRef` -> `$toRef`

### Optional Import

Because `$ref()` is a macro and not a runtime API, it doesn't need to be imported from `vue`. However, if you want to be more explicit, you can import it from `vue/macros`:

```js
import { $ref } from 'vue/macros'

let count = $ref(0)
```

## Bind existing ref as reactive variables with `$()`

In some cases we may have wrapped functions that also return refs. However, the Vue compiler won't be able to know ahead of time that a function is going to return a ref. Therefore, we also provide a more generic `$()` macro that can convert any existing refs into reactive variables:

```js
function myCreateRef() {
  return ref(0)
}

let count = $(myCreateRef())
```

## Destructuring objects of refs with `$()`

It is common for a composition function to return an object of refs, and use destructuring to retrieve these refs. `$()` can also be used in this case:

```js
import { useMouse } from '@vueuse/core'

const { x, y } = $(useMouse())

console.log(x, y)
```

Compiled output:

```js
import { toRef } from 'vue'
import { useMouse } from '@vueuse/core'

const __temp = useMouse(),
  x = toRef(__temp, 'x'),
  y = toRef(__temp, 'y')

console.log(x.value, y.value)
```

Note that if `x` is already a ref, `toRef(__temp, 'x')` will simply return it as-is and no additional ref will be created. If a destructured value is not a ref (e.g. a function), it will still work - the value will be wrapped into a ref so the rest of the code work as expected.

`$()` destructure works on both reactive objects AND plain objects containing refs.

## Reactive props destructure

There are two pain points with the current `defineProps()` usage in `<script setup>`:

1. Similar to `.value`, you need to always access props as `props.x` in order to retain reactivity. This means you cannot destructure `defineProps` because the resulting destructured variables are not reactive and will not update.

2. When using the [type-only props declaration](https://v3.vuejs.org/api/sfc-script-setup.html#typescript-only-features), there is no easy way to declare default values for the props. We introduced the `withDefaults()` API for this exact purpose, but it's still clunky to use.

We can address these issues by applying the same logic for reactive variables destructure to `defineProps`:

```html
<script setup lang="ts">
  interface Props {
    msg: string
    count?: number
    foo?: string
  }

  const {
    msg,
    // default value just works
    count = 1,
    // local aliasing also just works
    // here we are aliasing `props.foo` to `bar`
    foo: bar
  } = defineProps<Props>()

  watchEffect(() => {
    // will log whenever the props change
    console.log(msg, count, bar)
  })
</script>
```

the above will be compiled into the following runtime declaration equivalent:

```js
export default {
  props: {
    msg: { type: String, required: true },
    count: { type: Number, default: 1 },
    foo: String
  },
  setup(props) {
    watchEffect(() => {
      console.log(props.msg, props.count, props.foo)
    })
  }
}
```

## Retaining reactivity across function boundaries with `$$()`

While reactive variables relieve us from having to use `.value` everywhere, it creates an issue of "reactivity loss" when we pass reactive variables across function boundaries. This can happen in two cases:

### Passing into function as argument

Given a function that expects a ref object as argument, e.g.:

```ts
function trackChange(x: Ref<number>) {
  watch(x, (x) => {
    console.log('x changed!')
  })
}

let count = $ref(0)
trackChange(count) // doesn't work!
```

The above case will not work as expected because it compiles to:

```ts
let count = ref(0)
trackChange(count.value)
```

Here `count.value` is passed as a number where `trackChange` expects an actual ref. This can be fixed by wrapping `count` with `$$()` before passing it:

```diff
let count = $ref(0)
- trackChange(count)
+ trackChange($$(count))
```

The above compiles to:

```js
import { ref } from 'vue'

let count = ref(0)
trackChange(count)
```

As we can see, `$$()` is a macro that serves as an **escape hint**: reactive variables inside `$$()` will not get `.value` appended.

### Returning inside function scope

Reactivity can also be lost if reactive variables are used directly in a returned expression:

```ts
function useMouse() {
  let x = $ref(0)
  let y = $ref(0)

  // listen to mousemove...

  // doesn't work!
  return {
    x,
    y
  }
}
```

The above return statement compiles to:

```ts
return {
  x: x.value,
  y: y.value
}
```

In order to retain reactivity, we should be returning the actual refs, not the current value at return time.

Again, we can use `$$()` to fix this. In this case, `$$()` can be used directly on the returned object - any reference to reactive variables inside the `$$()` call will be retained as reference to their underlying refs:

```ts
function useMouse() {
  let x = $ref(0)
  let y = $ref(0)

  // listen to mousemove...

  // fixed
  return $$({
    x,
    y
  })
}
```

### `$$()` Usage on destructured props

`$$()` works on destructured props since they are reactive variables as well. The compiler will convert it with `toRef` for efficiency:

```ts
const { count } = defineProps<{ count: number }>()

passAsRef($$(count))
```

compiles to:

```js
setup(props) {
  const __props_count = toRef(props, 'count')
  passAsRef(__props_count)
}
```

<!--
### `$$()` Usage in templates

Refs are automatically unwrapped in Vue template expressions, and this has created a limitation where we previously don't have a way to pass raw ref objects down to child components as props.

The `$$()` macro will also be supported in template expressions so we are no longer subject to this limitation:

```html
<ChildComp :expect-ref="$$(someRef)"></ChildComp>
```
-->

## TypeScript & Tooling Integration

Vue will provide typings for these macros (available globally) and all types will work as expected. There are no incompatibilities with standard TypeScript semantics so the syntax would work with all existing tooling.

This also means the macros can work in any files where valid JS/TS are allowed - not just inside Vue SFCs.

Since the macros are available globally, their types need to be explicitly referenced (e.g. in a `env.d.ts` file):

```ts
/// <reference types="vue/macros-global" />
```

When explicitly importing the macros from `vue/macros`, the type will work without declaring the globals.

## Implementation Status

You can try the transform in the [Vue SFC Playground](https://sfc.vuejs.org/) (works in both `.vue` and `.(js|ts)` files).

Vue 3.2.25+ ships an implementation of this RFC as an experimental feature under the package [`@vue/reactivity-transform`](https://github.com/vuejs/vue-next/tree/master/packages/reactivity-transform). The package can be used standalone as a low-level library. It is also integrated (with its APIs re-exported) in `@vue/compiler-sfc` so most userland projects won't need to explicitly install it.

Higher-level tools like `@vitejs/plugin-vue` and `vue-loader` can be configured to apply the transform to vue, js(x) and ts(x) files. See [Appendix](#appendix) for how to enable the transform in specific tools.

> **Experimental features are unstable and may change between any release types (including patch releases). By explicitly enabling an experimental feature, you are taking on the risk of potentially having to refactor into updated syntax, or even refactor away from the usage if the feature ends up being removed.**

# Unresolved Questions

## `defineProps` destructure details

1. Should `defineProps` destructure require additional hints?

   Some may have the concern that reactive destructure of `defineProps` isn't obvious enough because it doesn't have the `$()` indication, which may confuse new users.

   An alternative of making it more explicit would be requiring `$()` to enable the reactive behavior:

   ```js
   const { foo } = $(defineProps(['foo']))
   ```

   However, the only benefit of this is for a new user to more easily notice that `foo` is reactive. If this change lands, the documentation would mention the destructure reactivity when introducing `defineProps`. Assuming all users learn about this on inital onboarding, the extra wrapping doesn't really serve any real purpose (similar to `$(ref(0))`).

2. The proposed reactive destructure for `defineProps` is technically a small breaking change, because previously the same syntax also worked, just without the reactivity. This could technically break the case where the user intentionally destructures the props object to get a non-reactive initial value of a prop:

   ```js
   const { foo } = defineProps(['foo'])
   ```

   However, this should be extremely rare because without reactive destructure, doing so meant **all** props retrieved this way are non-reactive. A more realistic example would be:

   ```js
   const props = defineProps(['foo', 'bar'])
   const { foo } = props
   props.bar // reactive access to `bar`
   ```

   A simple workaround after this RFC:

   ```js
   const { foo, bar } = defineProps(['foo', 'bar'])
   const initialFoo = foo
   ```

# Alternatives

## Other related proposals

This RFC is a revised version of [#368](https://github.com/vuejs/rfcs/pull/368) which also includes feedback from discussions in [#413](https://github.com/vuejs/rfcs/discussions/413) and [#394](https://github.com/vuejs/rfcs/discussions/394#discussioncomment-1391326).

### Earlier proposals

The whole discussion traces all the way back to the first draft of ref sugar, but most are outdated now. They are listed here for the records.

- https://github.com/vuejs/rfcs/issues/223 by @jods4
- https://github.com/vuejs/rfcs/pull/228 using label syntax
- https://github.com/vuejs/rfcs/pull/213
- https://github.com/vuejs/rfcs/pull/214

# Adoption strategy

This feature is opt-in. Existing code is unaffected.

# Appendix

## Enabling the Macros

- All setups require `vue@^3.2.25`

### Vite

- Requires `@vitejs/plugin-vue@^2.0.0`
- Applies to SFCs and js(x)/ts(x) files. A fast usage check is performed on files before applying the transform so there should be no performance cost for files not using the macros.
- Note `refTransform` is now a plugin root-level option instead of nested as `script.refSugar`, since it affects not just SFCs.

```js
// vite.config.js
export default {
  plugins: [
    vue({
      reactivityTransform: true
    })
  ]
}
```

### `vue-cli`

- Currently only affects SFCs
- requires `vue-loader@^17.0.0`

```js
// vue.config.js
module.exports = {
  chainWebpack: (config) => {
    config.module
      .rule('vue')
      .use('vue-loader')
      .tap((options) => {
        return {
          ...options,
          reactivityTransform: true
        }
      })
  }
}
```

### Plain `webpack` + `vue-loader`

- Currently only affects SFCs
- requires `vue-loader@^17.0.0`

```js
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          reactivityTransform: true
        }
      }
    ]
  }
}
```
