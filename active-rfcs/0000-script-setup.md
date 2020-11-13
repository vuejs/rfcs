- Start Date: 2020-10-28
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/rfcs/pull/182
- Implementation PR: https://github.com/vuejs/vue-next/pull/2532

# Summary

Introduce a new script type in Single File Components: `<script setup>`, which exposes all its top level bindings to the template.

**Note:** this is intended to replace the current `<script setup>` as proposed in [#182](https://github.com/vuejs/rfcs/pull/182).

# Basic example

```html
<script setup>
  // imported components are also directly usable in template
  import Foo from './Foo.vue'
  import { ref } from 'vue'

  // write Composition API code just like in a normal setup()
  // but no need to manually return everything
  const count = ref(0)
  const inc = () => {
    count.value++
  }
</script>

<template>
  <Foo :count="count" @click="inc" />
</template>
```

<details>
<summary>Compiled Output</summary>

```html
<script setup>
  import Foo from './Foo.vue'
  import { ref } from 'vue'

  export default {
    setup() {
      const count = ref(1)
      const inc = () => {
        count.value++
      }

      return {
        Foo, // see note below
        count,
        inc,
      }
    },
  }
</script>

<template>
  <Foo :count="count" @click="inc" />
</template>
```

**Note:** the SFC compiler also extracts binding metadata from `<script setup>` and use it during template compilation. This is why the template can use `Foo` as a component here even though it's returned from `setup()` instead of registered via `components` option.

</details>

# Motivation

This proposal's main goal is reducing the verbosity of Composition API usage inside Single File Components (SFCs) by directly exposing the context of `<script setup>` to the template.

We have a prior proposal for `<script setup>` [here](https://github.com/vuejs/rfcs/blob/sfc-improvements/active-rfcs/0000-sfc-script-setup.md), which is currently implemented (but marked as experimental). The old proposal opted for the `export` syntax so that the code would play well with unused variable checks.

This proposal takes a different direction based on the premise that we can offer customized linter rules in `eslint-plugin-vue`. This allows us to aim for the most succinct syntax possible.

# Detailed design

## `<script setup>`

To opt-in to the syntax, add the `setup` attribute to the `<script>` block:

```html
<script setup>
  // syntax enabled
</script>
```

### Top level bindings are exposed to template

Any top-level bindings (both variables and imports) declared inside `<script setup>` are directly exposed to the template render context:

```html
<script setup>
  import Foo from './Foo.vue'
  const msg = 'Hello!'
</script>

<template>
  <Foo>{{ msg }}</Foo>
</template>
```

<details>
<summary>Compiled Output</summary>

```html
<script>
  import Foo from './Foo.vue'

  export default {
    setup() {
      const msg = 'Hello!'

      return {
        Foo,
        msg,
      }
    },
  }
</script>

<template>
  <Foo>{{ msg }}</Foo>
</template>
```

**Note:** The SFC compiler also extracts binding metadata from `<script setup>` and use it during template compilation. Therefore in the template, `Foo` can be used as a component even though it's returned from `setup()` instead of registered via `components` option.

</details>
<p></p>

### Scoping mental model

Some users may be concerned that it is no longer clear what variables are exposed to the template vs. those that are not. However, it can be argued that there is no practical benefits in being able to tell this.

In the past we have always considered the `<script>` and `<template>` parts of an SFC to be separate parts that need an explicit protocol to be "connected". It does feel different to have this separation blurred. However, if we switch the scoping mental model to view an SFC as returning a render function from the `setup()` closure instead:

```js
function setup() {
  let a = 1
  let b = a + 1

  // the returned function has access to everything inside setup()
  // however it may or may not use all of them.
  return () => {
    return h('div', b)
  }
}
```

In fact, we have also introduced an [inline template](#inline-template-mode) option that will compile SFCs into this format. By inlining the generated render function inside `setup()` scope, we can directly access in scope variables without having to go through the render proxy. This can lead to decent performance gains.

This does require different handling when linting for unused variables, but SFCs already require the usage of `eslint-plugin-vue` which can be made to adapt to this new scoping model.

### Closed by default

In a Vue component, everything exposed to the template is implicitly exposed on the component instance, which can be retrieved by a parent component via template refs. That is to say, up to this point the **template render context** and the **imperative public interface** of a component is one and the same. We have found this to be problematic because the two use cases do not always align perfectly. In fact, most of the time we are over-exposing on the public interface front. This is why we are discussing an explicit way to define a component's imperative public interface in https://github.com/vuejs/rfcs/pull/210.

`<script setup>` as proposed in this RFC, if following current behavior, will be vastly over-exposing on the imperative public interface, therefore a component using `<script setup>` will be **closed by default**. That is to say, its public imperative interface will be an empty object unless bindings are explicitly exposed. How to explicitly expose imperative public interface will be finalized in https://github.com/vuejs/rfcs/pull/210.

### Declaring options and setup context

In order to declare options like `props` and `emits`, and also receive arguments that are passed to the `setup()` function, we can use the newly introduced `useOptions` API:

```html
<script setup>
  import { useOptions } from 'vue'

  const { props, emit, slots, attrs } = useOptions({
    props: {
      foo: String,
    },
    emits: ['change', 'delete'],
  })

  // setup code
</script>
```

The return value of `useOptions` is the setup context (2nd argument passed to `setup()`). Note that in current core API, the setup context does not expose `props` - in order to align the concepts, we are updating the setup context to expose `props` as well.

**Compiled output**

```html
<script>
  export default {
    props: {
      foo: String,
    },
    emits: ['change', 'delete'],
    setup(_, { props, emit, slots, attrs }) {
      // setup code
    },
  }
</script>
```

**Additional notes:**

- `useOptions` provides type inference similar to `defineComponent`.

- `useOptions` is a **compiler hint function**: it is compiled away when `<script setup>` is processed. The actual runtime implementation is a no-op and should never be called. Doing so will result in a runtime warning.

- The options passed to `useOptions` will be hoisted out of setup into module scope. Therefore, the options cannot reference local variables declared in setup scope. Doing so will result in a compile error. However, it _can_ reference imported bindings since they are in the module scope as well.

### Type-only props/emit declarations

Props and emits can also be declared using pure-type syntax by passing a literal type argument to `useOptions`:

```ts
const { props, emit } = useOptions<{
  props: {
    foo: string
    bar?: number
  }
  emit: (e: 'update' | 'delete', id: number) => void
}>()
```

- `useOptions` can only use either runtime declaration OR type declartion. Using both at the same time will result in a compile error.

- When using type declaration, equivalent runtime declaration is automatically generated from static analysis to remove the need of double declaration and still ensure correct runtime behavior.

  - In dev mode, the compiler will try to infer corresponding runtime validation from the types. For example here `foo: String` is inferred from the `foo: string` type. If the type is a reference to an imported type, the inferred result will be `foo: null` (equal to `any` type) since the compiler does not have information of external files.

  - In prod mode, the compiler will generate the array format declaration to reduce bundle size (the props here will be compiled into `['msg']`)

  - The emitted code is still TypeScript with valid typing, which can be further processed by other tools.

- As of now, the type declaration argument must be a literal type to ensure correct static analysis. This can be improved in the future.

### Top level await

Top level `await` can be used inside `<script setup>`. The resulting `setup()` function will be made `async`:

```html
<script setup>
  const post = await fetch(`/api/post/1`).then((r) => r.json())
</script>
```

<details>
<summary>Compiled Output</summary>

```html
<script>
  export default {
    async setup() {
      const post = await fetch(`/api/post/1`).then((r) => r.json())

      return { post }
    },
  }
</script>
```

</details>

## Usage alongside normal `<script>`

There are some cases where the code must be executed in the module scope, for example:

- Declaring named exports

- Global side effects that should only execute once.

In such cases, a normal `<script>` block can be used alongside `<script setup>`:

```html
<script>
  performGlobalSideEffect()

  // this can be imported as `import { named } from './*.vue'`
  export const named = 1
</script>

<script setup>
  let count = 0
</script>
```

<details>
<summary>Compile Output</summary>

```js
import { ref } from 'vue'

performGlobalSideEffect()

export const named = 1

export default {
  setup() {
    const count = ref(0)
    return {
      count,
    }
  },
}
```

</details>

## Usage restrictions

Due to the difference in module execution semantics, code inside `<script setup>` relies on the context of an SFC. When moved into external `.js` or `.ts` files, it may lead to confusions for both developers and tools. Therefore, **`<script setup>`** cannot be used with the `src` attribute.

# Drawbacks

## Tooling Compatiblity

This new scoping model will require tooling adjustments in two aspects:

1. ESLint rules like `no-unused-vars`. We will need a replacement rule in `eslint-plugin-vue` that takes both the `<script setup>` and `<template>` expressions into account.

2. Vetur and other IDEs will need to provide dedicated handling for this new `<script setup>` model in order to provide template expression type checking / props validation, etc.

One thing to note is that `<script setup>` syntax should make it easier for tools to analyze available bindings to the template compared to current Options API inside `<script setup>` (especially with mixins). A potential approach is transforming the entire SFC into a TSX file (similar to [svelte2tsx](https://github.com/sveltejs/language-tools/tree/master/packages/svelte2tsx)) for type-checking and refactoring support only.

# Alternatives

https://github.com/vuejs/rfcs/pull/182 (current `<script setup>` implementation)

# Adoption strategy

This feature is opt-in. Existing SFC usage is unaffected.

# Unresolved questions

This RFC depends on https://github.com/vuejs/rfcs/pull/210 being resolved first.

# Appendix

## Transform API

The `@vue/compiler-sfc` package exposes the `compileScript` method for processing `<script setup>`:

```js
import { parse, compileScript } from '@vue/compiler-sfc'

const descriptor = parse(`...`)

if (descriptor.script || descriptor.scriptSetup) {
  const result = compileScript(descriptor) // returns SFCScriptBlock
  console.log(result.code)
  console.log(result.bindings) // see next section
}
```

The compilation requires the entire descriptor to be provided, and the resulting code will include sources from both `<script setup>` and normal `<script>` (if present). It is the higher level tools' (e.g. `vite` or `vue-loader`) responsibility to properly assemble the compiled output.

## Inline Template Mode

Inline template mode can be enabled via the `inlineTemplate` option:

```js
compileScript(descriptor, { inlineTemplate: true })
```

This will compile the SFC template as well and inline it inside the `setup()` function generated from `<script setup>`. Example:

```html
<script setup>
  import { ref } from 'vue'

  const count = ref(0)

  function inc() {
    count.value++
  }
</script>
<template>
  <button @click="inc">{{ count }}</button>
</template>
```

**Compiled to:**

```js
import { ref, unref, createVNode, toDisplayString } from 'vue'

export default {
  setup() {
    const count = ref(0)

    function inc() {
      count.value++
    }

    return () => {
      createVNode(
        'div',
        {
          onClick: inc,
        },
        toDisplayString(unref(count))
      )
    }
  },
}
```

Note some bindings need to be wrapped with `unref` - the compiler performs some heuristics to avoid this when possible. For example, function declarations and const declarations with literal initial values will not be wrapped with `unref`.

## Template binding optimization

The `SFCScriptBlock` returned by `compiledScript` also exposes a `bindings` object, which is the exported binding metadata gathered during the compilation. For example, given the following `<script setup>`:

```vue
<script setup="props">
export const foo = 1

export default {
  props: ['bar'],
}
</script>
```

The `bindings` object will be:

```js
{
  foo: 'setup',
  bar: 'props'
}
```

This object can then be passed to the template compiler:

```js
import { compile } from '@vue/compiler-dom'

compile(template, {
  bindingMetadata: bindings,
})
```

With the binding metadata available, the template compiler can generate code that directly access template variables from the corresponding source, without having to go through the render context proxy:

```html
<div>{{ foo + bar }}</div>
```

```js
// code generated without bindingMetadata
// here _ctx is a Proxy object that dynamically dispatches property access
function render(_ctx) {
  return createVNode('div', null, _ctx.foo + _ctx.bar)
}

// code generated with bindingMetadata
// bypasses the render context proxy
function render(_ctx, _cache, $setup, $props, $data) {
  return createVNode('div', null, $setup.foo + $props.bar)
}
```

The binding information is also used in inline template mode to generate more efficient code.
