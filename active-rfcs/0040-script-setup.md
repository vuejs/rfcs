- Start Date: 2020-10-28
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/rfcs/pull/182
- Implementation PR: https://github.com/vuejs/vue-next/pull/2532

# Summary

Introduce a new script type in Single File Components: `<script setup>`, which exposes all its top level bindings to the template.

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

```js
import Foo from './Foo.vue'
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(1)
    const inc = () => {
      count.value++
    }

    return function render() {
      return h(Foo, {
        count,
        onClick: inc
      })
    }
  }
}
```

**Note:** the SFC compiler also extracts binding metadata from `<script setup>` and use it during template compilation. This is why the template can use `Foo` as a component here even though it's returned from `setup()` instead of registered via `components` option.

</details>
<p></p>

**Declaring Props and Emits**

```html
<script setup>
  // expects props options
  const props = defineProps({
    foo: String
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

When using `<script setup>`, the template is compiled into a render function that is inlined inside the setup function scope. This means any top-level bindings (both variables and imports) declared inside `<script setup>` are directly usable in the template:

```html
<script setup>
  const msg = 'Hello!'
</script>

<template>
  <div>{{ msg }}</div>
</template>
```

**Compiled Output:**

```js
export default {
  setup() {
    const msg = 'Hello!'

    return function render() {
      // has access to everything inside setup() scope
      return h('div', msg)
    }
  }
}
```

It is important to notice the different template scoping mental model vs. Options API: when using Options API, the `<script>` and the template are connected via a "render context object". When we write code, we are always thinking about "what properties are exposed on the context". This naturally leads to concerns of "leaking too much private logic onto the context".

When using `<script setup>`, however, the mental model is simply that of a function inside another function: the inner function has access to everything in the parent scope, and because the parent scope is closure, there is no "leak" to be concerned with.

### Using Components

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

```js
import Foo from './Foo.vue'
import MyComponent from './MyComponent.vue'

export default {
  setup() {
    return function render() {
      return [h(Foo), h(MyComponent)]
    }
  }
}
```

**Note**: in this case the template compiler has the binding information to generate code that directly use `Foo` from setup bindings instead of dynamically resolving it.

</details>
<p></p>

### Using Dynamic Components

Since components are referenced as variables instead of registered under string keys, we should use dynamic `:is` binding when using dynamic components inside `<script setup>`:

```html
<script setup>
  import Foo from './Foo.vue'
  import Bar from './Bar.vue'
</script>

<template>
  <component :is="Foo" />
  <component :is="someCondition ? Foo : Bar" />
</template>
```

<details>
<summary>Compiled Output</summary>

```js
import Foo from './Foo.vue'
import Bar from './Bar.vue'

export default {
  setup() {
    return function render() {
      return [h(Foo), h(someCondition ? Foo : Bar)]
    }
  }
}
```

</details>

### Using Directives

Directives work in a similar fashion - except that a directive named `v-my-dir` will map to a setup scope variable named `vMyDir`:

```html
<script setup>
  import { directive as vClickOutside } from 'v-click-outside'
</script>

<template>
  <div v-click-outside />
</template>
```

<details>
<summary>Compiled Output</summary>

```js
import { directive as vClickOutside } from 'v-click-outside'

export default {
  setup() {
    return function render() {
      return withDirectives(h('div'), [[vClickOutside]])
    }
  }
}
```

</details>

The reason for requiring the `v` prefix is because it is quite likely for a globally registered directive (e.g. `v-focus`) to clash with a locally declared variable of the same name. The `v` prefix makes the intention of using a variable as a directive more explicit and reduces unintended "shadowing".

### Declaring `props` and `emits`

To declare options like `props` and `emits` with full type inference support, we can use the `defineProps` and `defineEmits` APIs, which are automatically available inside `<script setup>`:

```html
<script setup>
  const props = defineProps({
    foo: String
  })

  const emit = defineEmits(['change', 'delete'])
  // setup code
</script>
```

<details>
<summary>Compiled output</summary>

```js
export default {
  props: {
    foo: String
  },
  emits: ['change', 'delete'],
  setup(props, { emit }) {
    // setup code
  }
}
```

</details>

- `defineProps` and `defineEmits` provides proper type inference based on the options passed.

- `defineProps` and `defineEmits` are **compiler macros** only usable inside `<script setup>`. They do not need to be imported, and are compiled away when `<script setup>` is processed.

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

### Default props values when using type declaration

One drawback of the type-only `defineProps` declaration is that it doesn't have a way to provide default values for the props. To resolve this problem, a `withDefaults` compiler macro is also provided:

```ts
interface Props {
  msg?: string
}

const props = withDefaults(defineProps<Props>(), {
  msg: 'hello'
})
```

This will be compiled to equivalent runtime props `default` options. In addition, the `withDefaults` helper provides type checks for the default values, and ensures the returned `props` type has the optional flags removed for properties that do have default values declared.

### Top level await

Top level `await` can be used inside `<script setup>`. The resulting `setup()` function will be made `async`:

```html
<script setup>
  const post = await fetch(`/api/post/1`).then((r) => r.json())
</script>
```

In addition, the awaited expression will be automatically compiled in a format that preserves the current component instance context after the `await`:

```js
import { withAsyncContext } from 'vue'

export default {
  async setup() {
    let __temp, __restore

    const post =
      (([__temp, __restore] = withAsyncContext(() =>
        fetch(`/api/post/1`).then((r) => r.json())
      )),
      (__temp = await __temp),
      __restore(),
      __temp)

    // current instance context preserved
    // e.g. onMounted() will still work.

    return { post }
  }
}
```

Relevant: https://github.com/vuejs/rfcs/issues/234

### Exposing component's public interface

In a traditional Vue component, everything exposed to the template is implicitly exposed on the component instance, which can be retrieved by a parent component via template refs. That is to say, up to this point the **template render context** and the **imperative public interface** of a component is one and the same. We have found this to be problematic because the two use cases do not always align perfectly. In fact, most of the time we are over-exposing on the public interface front. This is why we are discussing an explicit way to define a component's imperative public interface in the [Expose RFC](https://github.com/vuejs/rfcs/pull/210).

With `<script setup>` the template has access to the declared variables because it is compiled into a function that is returned from the `setup()` function scope. This means all the variables declared are in fact never returned: they are contained inside the `setup()` closure. As a result, a component using `<script setup>` will be **closed by default**. That is to say, its public imperative interface will be an empty object unless bindings are explicitly exposed.

To explicitly expose properties in a `<script setup>` component, use the `defineExpose` compiler macro:

```html
<script setup>
  const a = 1
  const b = ref(2)

  defineExpose({
    a,
    b
  })
</script>
```

When a parent gets an instance of this component via template refs, the retrieved instance will be of the shape `{ a: number, b: number }` (refs are automatically unwrapped just like on normal instances).

This will be compiled into the runtime equivalent as proposed in the [Expose RFC](https://github.com/vuejs/rfcs/pull/210).

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
  import { ref } from 'vue'

  const count = ref(0)
</script>
```

<details>
<summary>Compiled Output</summary>

```js
import { ref } from 'vue'

performGlobalSideEffect()

export const named = 1

export default {
  setup() {
    const count = ref(0)
    return {
      count
    }
  }
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

If you need to declare these options, use a separate normal `<script>` block with `export default`:

```html
<script>
  export default {
    name: 'CustomName',
    inheritAttrs: false,
    customOptions: {}
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

   As of now, [Volar](https://github.com/johnsoncodehk/volar) already provides full support for this RFC in VSCode, including all TypeScript related features. Its internals are also implemented as a language server that can theoretically be used in other IDEs.

2. ESLint rules like `no-unused-vars`. We will need a replacement rule in `eslint-plugin-vue` that takes both the `<script setup>` and `<template>` expressions into account.

# Adoption strategy

This feature is opt-in. Existing SFC usage is unaffected.

# Unresolved questions

- Type-only props/emits declarations currently do not support using externally imported types. This is useful when reusing base props type definitions across multiple components.

  The type inference already works as expected in Volar's IDE support, the limitation is purely in that `@vue/compiler-sfc` needs to know the props keys in order to generate the correct equivalent runtime declarations.

  This is technically possible if we implement the logic to follow along type imports, read, and parse the import source. However, this is more of an implementation scope problem and does not fundamentally affect how the RFC design behaves.

# Appendix

> The following sections are only for tooling authors that needs to support `<script setup>` in respective SFC tooling integrations.

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

## Inline vs. Non-Inline Mode

During development, `<script setup>` still compiles to returned object instead of inlined render function for two reasons:

1. Devtools inspection
2. Template hot-reloading (HMR)

Inline template mode is only used in production and can be enabled via the `inlineTemplate` option:

```js
compileScript(descriptor, { inlineTemplate: true })
```

In inline mode, some bindings (e.g. return values from `ref()` calls) need to be wrapped with `unref`:

```js
export default {
  setup() {
    const msg = ref('hello')

    return function render() {
      return h('div', unref(msg))
    }
  }
}
```

The compiler performs some heuristics to avoid this when possible. For example, function declarations and const declarations with literal initial values will not be wrapped with `unref`.

## Template binding optimization

The `SFCScriptBlock` returned by `compiledScript` also exposes a `bindings` object, which is the exported binding metadata gathered during the compilation. For example, given the following `<script setup>`:

```html
<script setup="props">
  export const foo = 1

  export default {
    props: ['bar']
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
  bindingMetadata: bindings
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
