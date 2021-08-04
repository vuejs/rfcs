- Start Date: 2021-07-16
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/rfcs/pull/182, https://github.com/vuejs/rfcs/pull/228

# Summary

Introduce a compiler-based syntax sugar for using refs without `.value`.

# Basic example

```html
<script setup>
  // declaring a variable that compiles to a ref
  let count = $ref(1)

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

- Declare reactive variables with `$ref`
- Declare reactively-derived variables with `$computed`
- Destructure reactive variables from an object with `$fromRefs`
- Get the raw ref object of a `$ref`-declared variable with `$raw`

## `$ref`

```js
let count = $ref(0)

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

Variables declared with `$ref` can be accessed or mutated just like normal variables - but it enables reactivity for these operations. The `$ref` serves as a hint for the compiler to append `.value` to all references to the variable.

**Notes**

- `$ref` is a compile-time macro and does not need to be imported.
- `$ref` can only be used with `let` because it would be pointless to declare a constant ref.
- `$ref` can also be used to create a variable-like binding for other ref types, e.g. `computed`, `shallowRef` or even `customRef`:

  ```js
  let count = $ref(0)

  const plusOne = $ref(computed(() => count + 1))
  console.log(plusOne)
  ```

## `$computed`

Because `computed` is so commonly used, it also has its dedicated macro:

```diff
- const plusOne = $ref(computed(() => count + 1))
+ const plusOne = $computed(() => count + 1)
```

## Destructuring with `$fromRefs`

It is common for a composition function to return an object of refs. To declare multiple ref bindings with destructuring, we can use the `$fromRefs` macro:

```js
const { x, y, method } = $fromRefs(useFoo())

console.log(x, y)
method()
```

Compiled Output:

```js
import { shallowRef } from 'vue'

const { x: __x, y: __y, method: __method } = useFoo()
const x = shallowRef(__x)
const y = shallowRef(__y)
const method = shallowRef(__method)

console.log(x.value, y.value)
method.value()
```

Note this works even if a property is not a ref: for example the `method` property is a plain function here - `shallowRef` will wrap it as an actual ref so that the rest of the code could work as expected.

## Accessing Raw Refs with `$raw`

In some cases we may need to access the underlying raw ref object of a `$ref`-declared variable. We can do that with the `$raw` macro:

```js
let count = $ref(0)

const countRef = $raw(count)

fnThatExpectsRef(countRef)
```

Compiled Output:

```js
let count = ref(0)

const countRef = count

fnThatExpectsRef(countRef)
```

Think of `$raw` as "do not append `.value` to anything passed to me, and return it". This means it can also be used on an object containing `$ref` variables:

```js
let x = $ref(0)
let y = $ref(0)

const coords = $raw({ x, y })

console.log(coords.x.value)
```

## TypeScript & Tooling Integration

Vue will provide typings for these macros (available globally) and all types will work as expected. There are no incompatibilities with standard TypeScript semantics so the syntax would work with all existing tooling.

## Ref Usage in Nested Function Scopes

> This section isn't implemented as of now

Technically, `$ref` doesn't have to be limited to root level scope and can be used anywhere `let` declarations can be used, including nested function scope:

```js
function useMouse() {
  let x = $ref(0)
  let y = $ref(0)

  function update(e) {
    x = e.pageX
    y = e.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return $raw({
    x,
    y
  })
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

This proposal is currently implemented in 3.2.0-beta as an experimental feature.

- It is currently only usable in `<script setup>`
- It currently only processes root-level variable declarations and **does not work in nested function scopes**
- It is disabled by default and must be explicitly enabled by passing the `refSugar: true` option to `@vue/compiler-sfc`. See [Appendix](#appendix) for how to enable it in specific tools.

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
