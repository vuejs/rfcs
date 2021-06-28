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
<p></p>

**Declaring Props and Emits**

```html
<script setup>
  import { defineProps, defineEmits } from 'vue'

  // expects props options
  const props = defineProps({
    foo: String,
  })
  // expects emits options
  const emit = defineEmits(['update', 'delete'])
</script>
```

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
  const msg = 'Hello!'
</script>

<template>
  <div>{{ msg }}</div>
</template>
```

<details>
<summary>Compiled Output</summary>

```html
<script>
  export default {
    setup() {
      const msg = 'Hello!'

      return {
        msg,
      }
    },
  }
</script>

<template>
  <div>{{ msg }}</div>
</template>
```

</details>
<p></p>

### Exposing Components and Directives

Values in the scope of `<script setup>` can also be used directly as custom component tag names, similar to how it works in JSX:

```html
<script setup>
  import Foo from './Foo.vue'
  import MyComponent from './MyComponent.vue'
</script>

<template>
  <Foo />
  <!-- kebab-case also works -->
  <my-component />
</template>
```

<details>
<summary>Compiled Output</summary>

```html
<script>
  import Foo from './Foo.vue'
  import MyComponent from './MyComponent.vue'

  export default {
    setup() {
      return {
        Foo,
        MyComponent,
      }
    },
  }
</script>

<template>
  <Foo />
  <my-component />
</template>
```

**Note**: in this case the template compiler has the binding information to generate code that directly use `Foo` from setup bindings instead of dynamically resolving it.

</details>
<p></p>

Directives work in a similar fashion - a directive named `v-my-dir` will map to a setup scope variable named `myDir`:

```html
<script setup>
  import { directive as clickOutside } from 'v-click-outside'
</script>

<template>
  <div v-click-outside />
</template>
```

<details>
<summary>Compiled Output</summary>

```html
<script>
  import { directive as clickOutside } from 'v-click-outside'

  export default {
    setup() {
      return {
        clickOutside,
      }
    },
  }
</script>

<template>
  <div v-click-outside />
</template>
```

</details>

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

### Declaring `props` and `emits`

To declare options like `props` and `emits` with full type inference support, we can use the `defineProps` and `defineEmits` APIs:

```html
<script setup>
  import { defineProps, defineEmits } from 'vue'

  const props = defineProps({
    foo: String,
  })

  const emit = defineEmits(['change', 'delete'])
  // setup code
</script>
```

<details>
<summary>Compiled output</summary>

```html
<script>
  export default {
    props: {
      foo: String,
    },
    emits: ['change', 'delete'],
    setup(props, { emit }) {
      // setup code
    },
  }
</script>
```

</details>

- `defineProps` and `defineEmits` provides proper type inference based on the options passed.

- `defineProps` and `defineEmits` are **compiler hints**. They are compiled away when `<script setup>` is processed. The actual runtime implementations are no-ops and should never be called. Doing so will result in a runtime warning.

- The options passed to `defineProps` and `defineEmits` will be hoisted out of setup into module scope. Therefore, the options cannot reference local variables declared in setup scope. Doing so will result in a compile error. However, it _can_ reference imported bindings since they are in the module scope as well.

### Using `slots` and `attrs`

Usage of `slots` and `attrs` inside `<script setup>` should be relatively rare, since you can access them directly as `$slots` and `$attrs` in the template. In the rare case where you do need them, use the `useSlots` and `useAttrs` helpers respectively:

```html
<script setup>
  import { useSlots, useAttrs } from 'vue'

  const slots = useSlots()
  const attrs = useAttrs()
</script>
```

`useSlots` and `useAttrs` are actual runtime functions that return the equivalent of `setupContext.slots` and `setupContext.attrs`. They can be used in normal composition API functions as well.

### Type-only props/emit declarations

Props and emits can also be declared using pure-type syntax by passing a literal type argument to `defineProps` or `defineEmits`:

```ts
const props = defineProps<{
  foo: string
  bar?: number
}>()

const emit = defineEmits<{
  (e: 'change', id: number): void
  (e: 'update', value: string): void
}>()
```

- `defineProps` or `defineEmits` can only use either runtime declaration OR type declaration. Using both at the same time will result in a compile error.

- When using type declaration, equivalent runtime declaration is automatically generated from static analysis to remove the need of double declaration and still ensure correct runtime behavior.

  - In dev mode, the compiler will try to infer corresponding runtime validation from the types. For example here `foo: String` is inferred from the `foo: string` type. If the type is a reference to an imported type, the inferred result will be `foo: null` (equal to `any` type) since the compiler does not have information of external files.

  - In prod mode, the compiler will generate the array format declaration to reduce bundle size (the props here will be compiled into `['msg']`)

  - The emitted code is still TypeScript with valid typing, which can be further processed by other tools.

- As of now, the type declaration argument must be one of the following to ensure correct static analysis:

  - A type literal
  - A reference to a an interface or a type literal in the same file

  Currently complex types and type imports from other files are not supported. It is theoretically possible to support type imports in the future.

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

<p></p>

Relevant: https://github.com/vuejs/rfcs/issues/234

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

## Automatic `name` Inference

Vue 3 SFCs automatically infers the component's name from the component's **filename** in the following cases:

- Dev warning formatting
- DevTools inspection
- Recursive self-reference. E.g. a file named `FooBar.vue` can refer to itself as `<FooBar/>` in its template.

  This has lower priority than explicity registered/imported components. If you have a named import that conflicts with the component's inferred name, you can alias it:

  ```js
  import { FooBar as FooBarChild } from './components'
  ```

In most cases, explicit `name` declaration is not needed. The only cases where you do need it is when you need the `name` for `<keep-alive>` inclusion / exclusion or direct inspection of the component's options.

## Declaring Additional Options

The `<script setup>` syntax provides the ability to express equivalent functionality of most existing Options API options except for a few:

- `name`
- `inheritAttrs`
- Custom options needed by plugins or libraries

If you need to delcare these options, use a separate normal `<script>` block with `export default`:

```html
<script>
  export default {
    name: 'CustomName',
    inheritAttrs: false,
    customOptions: {},
  }
</script>

<script setup>
  // script setup logic
</script>
```

## Usage restrictions

Due to the difference in module execution semantics, code inside `<script setup>` relies on the context of an SFC. When moved into external `.js` or `.ts` files, it may lead to confusions for both developers and tools. Therefore, **`<script setup>`** cannot be used with the `src` attribute.

# Drawbacks

## Tooling Compatiblity

This new scoping model will require tooling adjustments in two aspects:

1. IDEs will need to provide dedicated handling for this new `<script setup>` model in order to provide template expression type checking / props validation, etc.

    As of now, [Volar](https://github.com/johnsoncodehk/volar) already provides full support for this RFC in VSCode, including all TypeScript related features. Its internals are also implemented as a landuage server that can theoretically be used in other IDEs.

2. ESLint rules like `no-unused-vars`. We will need a replacement rule in `eslint-plugin-vue` that takes both the `<script setup>` and `<template>` expressions into account.

# Adoption strategy

This feature is opt-in. Existing SFC usage is unaffected.

# Unresolved questions

- Providing props default values when using type-only props declaration.
- This RFC depends on https://github.com/vuejs/rfcs/pull/210.

# Appendix


## FAQs

### How to tell which properties are exposed to the template?

You don't. An important mental model shift when using `<script setup>` is to think of the `<template>` as **a function inside the setup scope** instead of "bound to a `this` context":

```js
function setup() {
  const a = 1
  const b = 2

  return function template() {
    // has access to `b` but doesn't necessarily uses it
    return `<div>${a}</div>`
  }
}
```

A function inside another function naturally has access to everything declared within the parent function's scope. The parent scope is a closure and it doesn't leak the variables to anything but the inner function. This is also why `<script setup>` components are closed by default: it won't expose anything on its ref instance unless explicitly declared.

### How to declare options like `name`?

See [Declaring Additional Options](#declaring-additional-options) and [Automatic Name Inference](#automatic-name-inference).

### How to use imported components as dynamic components?

Within `<script setup>`, imported components are **variables** instead of a registered asset looked up using string keys. So when using imported components as dynamic components, instead of `<component is="Foo">`, it should be `<component :is="Foo"/>`. You can also use these variables in expressions, e.g. `<component :is="ok ? Foo : Bar"/>`

### How to use render functions w/ script setup?

You can't. But you don't need to use SFC if you are using render functions in the first place.

```js
const comp = defineComponent({
  setup() {
    const foo = ref(1)
    // return render function w/ JSX
    return () => <div>{foo.value}</div>
  }
})
```


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
  foo: 'setup-const',
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
