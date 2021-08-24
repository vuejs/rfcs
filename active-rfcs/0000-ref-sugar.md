- Start Date: 2021-07-16
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/rfcs/pull/182, https://github.com/vuejs/rfcs/pull/228

# Summary

Introduce a set of compiler macros for using refs without `.value`.

# Basic example

```ts
// declaring a reactive variable backed by an underlying ref
let count = $ref(1)

// no need for .value anymore!
console.log(count) // 1

function inc() {
  // assignments are reactive
  count++
}
```

<details>
<summary>Compiled Output</summary>

```js
import { ref } from 'vue'

const count = ref(1)

console.log(count.value)

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

- Declare reactive variables from refs using `$()` (`refs -> vars`)
- Get the underlying refs from reactive variables with `$$()` (`vars -> refs`)
- Most commonly used APIs have convenience aliases (`$ref`, `$computed` & `$shallowRef`)

## Bind refs as reactive variables with `$()`

The `$ref(0)` usage seen in the basic example is in fact a shorthand for `$(ref(0))`. The example is equivalent to:

```js
import { ref } from 'vue'

let count = $(ref(0))

function inc() {
  count++
}
```

<details>
<summary>Compiled Output</summary>

```js
import { ref } from 'vue'

let count = ref(0)

function inc() {
  count.value++
}
```

</details>

By wrapping a ref with `$()`, the resulting varaible is what we call a **reactive variable**. It can be accessed or mutated just like normal variables - except the access and mutations are reactive.

In the above example, accessing `count` will access `.value` on the underlying ref, and assigning a new value to `count` will mutate `.value` of the underlying ref.

From the implementation perspecitve, `$()` is a marker that instructs the compiler to auto-append `.value` to all references to the declared variables.

- `$()` is a compile-time macro and does not need to be imported.
- `$()` can only be used with `let` because it would be pointless to declare a constant ref.
- `$()` can be used with any Vue Reactivity APIs that return refs:

```js
import { ref, computed, shallowRef, customRef, toRef } from 'vue'

let count = $(ref(0))
let plusOne = $(computed(() => count + 1))
let shallowValue = $(shallowRef({ ... }))
let custom = $(customRef({ ... }))
let pick = $(toRef(someObject, 'foo'))
```

## Frequent API Shorthands

Because some APIs are so frequently used, they have dedicated aliases to make the code more succinct (which also save the need for imports):

- `$ref` is alias for `$(ref())`
- `$computed` is alias for `$(computed())`
- `$shallowRef` is alias for `$(shallowRef())`

## Destructuring objects of refs

It is common for a composition function to return an object of refs, and use destructuring to retrive these refs. `$()` can be used in this case as well:

```js
import { useMouse } from '@vueuse/core'

let { x, y } = $(useMouse())

console.log(x, y)
```

<details>
<summary>Compiled Output</summary>

```js
import { ref } from 'vue'
import { useMouse } from '@vueuse/core'

let { x: __x, y: __y } = useMouse()
const x = ref(__x)
const y = ref(__y)

console.log(x.value, y.value)
```

Note that if `x` is already a ref, `ref(__x)` will simply return it as-is. This works becuase `ref()` will return its argument as-is if it's already a ref.

If a destructured value is not a ref (e.g. a function), it will still work - the value will be wrapped into a ref so the rest of the code work as expected.

</details>

## Retrieving refs from reactive variables with `$$()`

While reactive variables relieve us from having to use `.value` everywhere, it creates an issue of "reactivity loss" when we pass reactive varaibles across function boundaries. This can happen in two cases:

1. A function that expects a ref object as argument, e.g.:

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

2. When returning an object of refs from a composable function. Example:

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

   > Note: support for usage inside nested function scopes is not yet implemented as of now.

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

## TypeScript & Tooling Integration

Vue will provide typings for these macros (available globally) and all types will work as expected. There are no incompatibilities with standard TypeScript semantics so the syntax would work with all existing tooling.

This also means the macros can work in any files where valid JS/TS are allowed - not just inside Vue SFCs.

Since the macros are available globally, their types need to be explicitly referenced (e.g. in a `evn.d.ts` file):

```ts
/// <reference types="vue/ref-macros" />
```

## Implementation Status

You can try the transform in the [Vue SFC Playground](https://sfc.vuejs.org/) (works in both `.vue` and `.(js|ts)` files).

Vue 3.2.5+ ships an implementation of this RFC as an experimental feature under the package [`@vue/ref-transform`](https://github.com/vuejs/vue-next/tree/master/packages/ref-transform). The package can be used standalone as a low-level library. It is also integrated (with its APIs re-exported) in `@vue/compiler-sfc` so most userland projects won't need to explicitly install it.

Higher-level tools like `@vitejs/plugin-vue` and `vue-loader` can be configured to apply the transform to vue, js(x) and ts(x) files. See [Appendix](#appendix) for how to enable the transform in specific tools.

> **Experimental features are unstable and may change between any release types (including patch releases). By explicitly enabling an experimental feature, you are taking on the risk of potentially having to refactor into updated syntax, or even refactor away from the usage if the feature ends up being removed.**

# Unresolved Questions

N/A

# Alternatives

## Other related proposals

- https://github.com/vuejs/rfcs/issues/223 by @jods4
- https://github.com/vuejs/rfcs/pull/228 using label syntax
- https://github.com/vuejs/rfcs/pull/213
- https://github.com/vuejs/rfcs/pull/214

# Adoption strategy

This feature is opt-in. Existing code is unaffected.

# Appendix

## Enabling the Macros

- All setups require `@vue/compiler-sfc@^3.2.5`

### Vite

- Requires `@vitejs/plugin-vue@^1.6.0`
- Applies to SFCs and js(x)/ts(x) files. A fast usage check is performed on files before applying the transform so there should be no performance cost for files not using the macros.
- Note `refTransform` is now a plugin root-level option instead of nested as `script.refSugar`, since it affects not just SFCs.

```js
// vite.config.js
export default {
  plugins: [
    vue({
      refTransform: true
    })
  ]
}
```

### `vue-cli`

- Currently only affects SFCs

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
          refSugar: true
        }
      })
  }
}
```

### Plain `webpack` + `vue-loader`

- Currently only affects SFCs

```js
// webpack.config.js
module.exports = {
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader',
        options: {
          refSugar: true
        }
      }
    ]
  }
}
```
