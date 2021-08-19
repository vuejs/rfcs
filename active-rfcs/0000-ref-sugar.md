- Start Date: 2021-07-16
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/rfcs/pull/182, https://github.com/vuejs/rfcs/pull/228

# Summary

Introduce a compiler-based syntax sugar for using refs without `.value`.

# Basic example

```html
<script setup>
  // declaring a variable that compiles to a ref
  let count = $(1)

  console.log(count) // 1

  function inc() {
    // the variable can be used like a plain value
    count++
  }
</script>

<template>
  <button @click="inc">{{ count }}</button>
</template>
```

<details>
<summary>Compiled Output</summary>

```js
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(1)

    console.log(count.value)

    function inc() {
      count.value++
    }

    return () => {
      return h('button', { onClick: inc }, count.value)
    }
  }
}
```

</details>

# Motivation

Ever since the introduction of the Composition API, one of the primary unresolved questions is the use of refs vs. reactive objects. It can be cumbersome to use `.value` everywhere, and it is easy to miss if not using a type system. Some users specifically lean towards using `reactive()` exclusively so that they don't have to deal with refs.

This proposal aims to improve the ergonomics of refs with a set of compile-time macros.

# Detailed design

## Overview

- Declare reactive variables with `$`
- Declare reactively-derived variables with `$computed`
- Get the raw ref object of a `$`-declared variable with `$ref`

## `$`

```js
let count = $(0)

function inc() {
  count++
}
```

Compiled Output:

```js
let count = ref(0)

function inc() {
  count.value++
}
```

Variables declared with `$` can be accessed or mutated just like normal variables - but it enables reactivity for these operations. `$` serves as a hint for the compiler to append `.value` to all references to the variable.

Note that:

- `$` is a compile-time macro and does not need to be imported.
- `$` can only be used with `let` because it would be pointless to declare a constant ref.

## Dereferencing Existing Refs

`$` can also be used to create reactive variables from existing refs, including refs returned from `computed`, `shallowRef`, or `customRef`. This is a bit similar to the concept of "dereferencing" in languages like C++, where the value is retrived from a memory pointer (in our case, a Vue ref object).

```js
let c = $(computed(() => count + 1))
console.log(c)
```

Compiled output:

```js
let c = ref(computed(() => count.value + 1))
console.log(c.value)
```

This works becuase `ref()` will return its argument as-is if it is already a ref:

```js
const foo = ref(0)
const bar = ref(foo)
console.log(foo === bar) // true
```

## `$computed`

Because `computed` is so commonly used, it also has its dedicated macro:

```diff
- let plusOne = $ref(computed(() => count + 1))
+ let plusOne = $computed(() => count + 1)
```

## Dereferencing + Destructuring

It is common for a composition function to return an object of refs, and use destructuring to retrive these refs. `$` can be used in such cases as well:

```js
let { x, y, method } = $(useFoo())
console.log(x, y)
method()
```

Compiled Output:

```js
import { ref } from 'vue'

let { x: __x, y: __y, method: __method } = useFoo()
const x = ref(__x)
const y = ref(__y)
const method = ref(__method)

console.log(x.value, y.value)
method.value()
```

Here if `__x` is already a ref, `ref(__x)` will simply return it as-is. Again, this works becuase `ref()` will return any existing ref as-is.

If the value is not a ref (e.g. `__method` which is a function), it will be normalized into an actual ref so the rest of the code work as expected. This is because we do not have the information to determine whether a destructured property is a ref at compile time.

## Accessing Raw Refs with `$ref`

In some cases we may need to access the underlying raw ref object of a reactive varaible. This typically happens when we have an external composable function that expects an actual ref as its argument. We can do this with the `$ref` macro:

```js
let count = $(0)

//`countRef` is the underlying ref object that syncs with `count`
const countRef = $ref(count)

countRef.value++
console.log(count) // 1

fnThatExpectsRef(countRef)
```

Compiled Output:

```js
let count = ref(0)

const countRef = count

countRef.value++
console.log(count.value) // 1

fnThatExpectsRef(countRef)
```

If you are familiar with languages like C++, this is also a bit similar to the "address-of" operator (`&`). However instead of the underlying memory address, we are getting a raw ref object instead.

## TypeScript & Tooling Integration

Vue will provide typings for these macros (available globally) and all types will work as expected. There are no incompatibilities with standard TypeScript semantics so the syntax would work with all existing tooling.

## Usage in Nested Function Scopes

> This section isn't implemented as of now

Technically, ref sugar doesn't have to be limited to root level scope and can be used anywhere `let` declarations can be used, including nested function scope:

```js
function useMouse() {
  let x = $(0)
  let y = $(0)

  function update(e) {
    x = e.pageX
    y = e.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return {
    x: $ref(x),
    y: $ref(y)
  }
}
```

<details>
<summary>Compiled Output</summary>

```js
function useMouse() {
  let x = ref(0)
  let y = ref(0)

  function update(e) {
    x.value = e.pageX
    y.value = e.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return {
    x,
    y
  }
}
```

</details>
<p></p>

## Implementation Status

This proposal is currently implemented in 3.2.0 as an experimental feature.

- It is currently only usable in `<script setup>`
- It currently only processes root-level variable declarations and **does not work in nested function scopes**
- It is disabled by default and must be explicitly enabled by passing the `refSugar: true` option to `@vue/compiler-sfc`. See [Appendix](#appendix) for how to enable it in specific tools.

**Experimental features are unstable and may change between any release types (including patch releases). By explicitly enabling an experimental feature, you are taking on the risk of potentially having to refactor into updated syntax, or even refactor away from the usage if the feature ends up being removed.**

# Unresolved Questions

## Should the macros be supported outside of SFCs?

The transforms proposed in this RFC can technically be applied to any JS/TS code via Babel or other AST transforms. It is currently implemented with support only inside `<script setup>` as part of `<script setup>` compilation because:

- Despite being syntactically valid JS/TS, "reactive variable assignment" is not standard JS semantics. `<script setup>` serves as an indication that the code is being pre-processed for special behavior.
- Since it is implemented as part of `@vue/compiler-sfc`, it allows existing Vue users to start using the syntax without any additional configuration.
- `<script setup>` compilation already performs full AST parsing, so the ref sugar transform can reuse the same AST and avoid incurring additional parsing overhead.
- The transform also integrates with the binding analysis (which is used by the template compiler for for optimized output).

### If Limited to SFCs

The primary drawback of limiting the syntax to SFCs is the mental model shift cost when writing code in/out of components. [A previous study](https://github.com/vuejs/rfcs/pull/222#issuecomment-723560606) shows that this mental cost may actually reduce efficiency compared to usage without the macros.

Differnet syntax also creates friction when it comes to extracting and reusing logic across components.

Luckily, since the transformation rules are relatively straightforward, code written with the sguar can be automatically de-sugared via tooling (e.g. via IDE extensions like Volar).

The workflow of extracting sguar code into an external composition function could be:

1. Select code range for the code to be extracted
2. In VSCode command input: `>volar de-sugar as composable' -> enter function name
3. Code gets de-sugared into a composable function (with return statements auto generated)
4. Cut-paste code into external file
5. Import the function in original file and replace original code.

### If Supported in All Files

There are some downsides in supporting them in all files:

- Transpile cost: we already parse and transform `<script setup>`, so ref macros do not add noticeable additional cost. However if applied to all JS/TS files, the cost can be much more significant.

- "Assignment-based reactivity" isn't standard JavaScript semantics and it may be a bad idea to let it leak outside of a Vue-specific context.

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

> Requires Vue >= 3.2.0-beta.1

### Vite

```js
// vite.config.js
export default {
  plugins: [
    vue({
      script: {
        refSugar: true
      }
    })
  ]
}
```

### `vue-cli`

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
