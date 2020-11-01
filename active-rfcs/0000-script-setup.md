- Start Date: 2020-10-28
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/rfcs/pull/182
- Implementation PR: https://github.com/vuejs/vue-next/pull/2532

# Summary

- Introduce a new script type in Single File Components: `<script setup>`, which exposes all its top level bindings to the template.

- Introduce a compiler-based syntax sugar for using refs without `.value` inside `<script setup>`.

- **Note:** this is intended to replace the current `<script setup>` as proposed in [#182](https://github.com/vuejs/rfcs/pull/182).

# Basic example

## 1. `<script setup>` now directly exposes top level bindings to template

```html
<script setup>
// imported components are also directly usable in template
import Foo from './Foo.vue'
import { ref } from 'vue'

// write Composition API code just like in a normal setup()
// but no need to manually return everything
const count = ref(0)
const inc = () => { count.value++ }
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
    const inc = () => { count.value++ }

    return {
      Foo, // see note below
      count,
      inc
    }
  }
}
</script>

<template>
  <Foo :count="count" @click="inc" />
</template>
```

**Note:** the SFC compiler also extracts binding metadata from `<script setup>` and use it during template compilation. This is why the template can use `Foo` as a component here even though it's returned from `setup()` instead of registered via `components` option.
</details>

## 2. `ref:` sugar makes ref usage more succinct

```html
<script setup>
// declaring a variable that compiles to a ref
ref: count = 1

function inc() {
  // the variable can be used like a plain value
  count++
}

// access the raw ref object by prefixing with $
console.log($count.value)
</script>

<template>
  <button @click="inc">{{ count }}</button>
</template>
```

<details>
<summary>Compiled Output</summary>

```html
<script setup>
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(1)

    function inc() {
      count.value++
    }

    console.log(count.value)

    return {
      count,
      inc
    }
  }
}
</script>

<template>
  <button @click="inc">{{ count }}</button>
</template>
```
</details>

# Motivation

This proposal has two main goals:

1. Reduce verbosity of Single File Component `<script>` by directly exposing its context to the template.

    We have a prior proposal for `<script setup>` [here](https://github.com/vuejs/rfcs/blob/sfc-improvements/active-rfcs/0000-sfc-script-setup.md), which is currently implemented (but marked as experimental). The old proposal opted for the `export` syntax so that the code would play well with unused variable checks.

    This proposal takes a different direction based on the premise that we can offer customized linter rules in `eslint-plugin-vue`. This allows us to aim for the most succinct syntax possible.

2. Improve ergonomics of refs with the `ref:` syntax sugar.

    Ever since the introduction of the Composition API, one of the primary unresolved questions is the use of refs vs. reactive objects. It can be cumbersome to use `.value` everywhere, and it is easy to miss if not using a type system. Some users specifically lean towards using `reactive()` exclusively so that they don't have to deal with refs.

    The existence of ref is mostly a design trade-off due to the constraints of the language we are working with: JavaScript. JavaScript does not provide a native way to pass reactive bindings around without wrapping it with an object. This means that **it is impossible to use refs like normal variable bindings without altering or augmenting JavaScript semantics.**

    - There has been a [proposal for adding native refs to JavaScript](https://github.com/rbuckton/proposal-refs), but it was designed to address a slightly different problem and doesn't seem to have received much attention.

    - A prominent example of altering JavaScript semantics in return for succinct syntax is [Svelte](https://svelte/). It [appropriates a number of JavaScript syntax to express framework-specific behavior](#svelte-syntax-details).

    In the past, we have tried to stick to strict JavaScript semantics as much as possible. Deviating from standard JavaScript semantics has number of [drawbacks](](#drawbacks)), but we believe there is room for a pragmatic trade-off where "breaking out of the box" a little bit can result in substantial improvements in developer experience.

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
      msg
    }
  }
}
</script>

<template>
  <Foo>{{ msg }}</Foo>
</template>
```

**Note:** The SFC compiler also extracts binding metadata from `<script setup>` and use it during template compilation. Therefore in the template, `Foo` can be used as a component even though it's returned from `setup()` instead of registered via `components` option.
</details>

### Setup Signature

The value of the `setup` attribute will be used as the arguments of the `setup()` function:

```html
<script setup="props, { emit }">
console.log(props.msg)
emit('foo')
</script>
```

<details>
<summary>Compiled Output</summary>

```html
<script>
export default {
  setup(props, { emit }) {
    console.log(props.msg)
    emit('foo')
  }
}
</script>
```
</details>

### Declaring Component Options

`export default` can still be used inside `<script setup>` for declaring component options such as props. Note that the exported expression will be hoisted out of `setup()` scope so it won't be able to reference variables declared in `<script setup>` (a compile error will be emitted in this case).

```html
<script setup="props">
export default {
  props: {
    msg: String
  }
}

console.log(props.msg)
</script>
```

<details>
<summary>Compiled Output</summary>

```html
<script>
export default {
  ...({
    props: {
      msg: String
    }
  }),
  setup(props) {
    console.log(props.msg)
  }
}
</script>
```
</details>

### Top level await

Top level `await` can be used inside `<script setup>`. The resulting `setup()` function will be made `async`:

```html
<script setup>
const post = await fetch(`/api/post/1`).then(r => r.json())
</script>
```

<details>
<summary>Compiled Output</summary>

```html
<script>
export default {
  async setup() {
    const post = await fetch(`/api/post/1`).then(r => r.json())

    return { post }
  }
}
</script>
```
</details>

## Ref Syntax

Code inside `<script setup>` can use a special `ref:` declaration to declare variables that can be used as a normal variable, but are compiled into refs:

```html
<script setup>
ref: count = 0

function inc() {
  count++
}
</script>

<template>
  <button @click="inc">{{ count }}</button>
</template>
```

`ref: count = 0` is a [labeled statement](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/label) which is syntactically valid in both JS/TS. However, we are using it as a variable declaration here. The compiler will:

1. Convert it to a proper variable declaration
2. Wrap its initial value with `ref()`
3. Rewrite all references to `count` into `count.value`.

<details>
<summary>Compiled Output</summary>

```js
import { ref } from 'vue'

export default {
  setup() {
    const count = ref(0)

    function inc() {
      count.value++
    }

    return {
      count,
      inc
    }
  }
}
```
</details>
<p></p>

Note that the syntax is opt-in: all Composition APIs can be used inside `<script setup>` without ref sugar:

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

### Accessing Raw Ref

It is common for an external composition function to expect a raw ref object as argument, so we need a way to access the raw underlying ref object for bindings declared via `ref:`. To deal with that, every `ref:` binding will have a corresponding `$`-prefixed counter part that exposes the raw ref:

```js
ref: count = 1
console.log($count.value) // 1

$count.value++
console.log(count) // 2

watch($count, newCount => {
  console.log('new count is: ', newCount)
})
```

<details>
<summary>Compiled Output</summary>

```js
const count = ref(1)
console.log(count.value) // 1

count.value++
console.log(count.value) // 2

watch(count, newCount => {
  console.log('new count is: ', newCount)
})
```
</details>

### Interaction with Non-Literals

`ref:` will wrap assignment values with `ref()`. If the value is already a ref, it will be returned as-is. This means we can use `ref:` with any function that returns a ref, for example `computed`:

```js
import { computed } from 'vue'

ref: count = 0
ref: plusOne = computed(() => count + 1)
console.log(plusOne) // 1
```

<details>
<summary>Compiled Output</summary>

```js
import { computed, ref } from 'vue'

const count = ref(0)
// `ref()` around `computed()` is a no-op here since return value
// from `computed()` is already a ref.
const plusOne = ref(computed(() => count.value + 1))
```
</details>
<p></p>

Or, any custom composition function that returns a ref:

```js
import { useMyRef } from './composables'

ref: myRef = useMyRef()
console.log(myRef) // no need for .value
```

<details>
<summary>Compiled Output</summary>

```js
import { useMyRef } from './composables'
import { ref } from 'vue'

// if useMyRef() returns a ref, it will be untouched
// otherwise it's wrapped into a ref
const myRef = ref(useMyRef())
console.log(myRef.value)
```
</details>
<p></p>

**Note:** if using TypeScript, this behavior creates a typing mismatch which we will discuss in [TypeScript Integration](#typescript-integration) below.

### Destructuring

It is common for a composition function to return an object of refs. To declare multiple ref bindings with destructuring, we can use [Destructuring Assignment](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment):

```js
ref: ({ x, y } = useMouse())
```

<details>
<summary>Compiled Output</summary>

```js
import { ref } from 'vue'

const { x: __x, y: __x } = useMouse()
const x = ref(__x)
const y = toRef(__y)
```
</details>
<p></p>

**Note:** object destructuring must be wrapped in parens - this is JavaScript's own syntax requirement to avoid ambiguity with a block statement.

## TypeScript Integration

### Typing props, slots, and emit

To type setup arguments like `props`, `slots` and `emit`, simply declare them:

```html
<script setup="props, { emit, slots }" lang="ts">
import { VNode } from 'vue'

// declare props using TypeScript syntax
// this will be auto compiled into runtime equivalent!
declare const props: {
  msg: string
}

// declare allowed emit signatures via overload
declare function emit(e: 'add', msg: string): void
declare function emit(e: 'remove', id: number): void

// you can even declare slot types
declare const slots: {
  default: () => VNode[]
}

emit('add', props.msg)
</script>
```

Runtime props and emits declaration is automatically generated from TS typing to remove the need of double declaration and still ensure correct runtime behavior. Note that the `props` type declaration value cannot be an imported type, because the SFC compiler does not process external files to extract the prop names.

<details>
<summary>Compile Output</summary>

```html
<script lang="ts">
import { VNode, defineComponent } from 'vue'

declare function __emit__(e: 'add', msg: string): void
declare function __emit__(e: 'remove', id: number): void

declare const slots: {
  default: () => VNode[]
}

export default defineComponent({
  // runtime declaration for props
  props: {
    msg: { type: String, required: true }
  } as unknown as undefined,

  // runtime declaration for emits
  emits: ["add"] as unknown as undefined,

  setup(props: { msg: string }, { emit }: {
    emit: typeof __emit__,
    slots: slots,
    attrs: Record<string, any>
  }) {
    emit('add', props.msg)
    return {}
  }
})
</script>
```

Details on runtime props generation:

- In dev mode, the compiler will try to infer corresponding runtime validation from the types. For example here `msg: String` is inferred from the `msg: string` type.

- In prod mode, the compiler will generate the array format declaration to reduce bundle size (the props here will be compiled into `['msg']`)

- The generated props declaration is force casted into `undefined` to ensure the user provided type is used in the emitted code.

- The emitted code is still TypeScript with valid typing, which can be further processed by other tools.
</details>

### Ref Declaration and Raw Ref Access

Unlike normal variable declarations, the `ref:` syntax has some special behavior in terms of typing:

- The declared variable always has the raw value type, regardless of whether the assigned value is a `Ref` type or not (always unwraps)

- The accompanying `$`-prefixed raw access variable always has a `Ref` type. If the right hand side value type already extends `Ref`, it will be used as-is; otherwise it will be wrapped as `Ref<T>`.

The following table demonstrates the resulting types of different usage:

| source | resulting type for `count` | resulting type for `$count` |
|--------|----------------------------|-----------------------------|
|`ref: count = 1`|`number`|`Ref<number>`|
|`ref: count = ref(1)`|`number`|`Ref<number>`|
|`ref: count = computed(() => 1)`|`number`|`ComputedRef<number>`|
|`ref: count = computed({ get:()=>1, set:_=>_ })`|`number`|`WritableComputedRef<number>`|

How to support this in Vetur is [discussed in the appendix](#ref-typescript-support-implementation-details).

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
      count
    }
  }
}
```
</details>

## Usage restrictions

Due to the difference in module execution semantics, code inside `<script setup>` relies on the context of an SFC. When moved into external `.js` or `.ts` files, it may lead to confusions for both developers and tools. Therefore, **`<script setup>`** cannot be used with the `src` attribute.

# Drawbacks

## Non-standard semantics

Some users may have strong aversion against non-standard semantics in their code. Many of the Vue team members held such concerns as well. However, consider that:

- Single file components look like HTML but isn't actually HTML. It already has its own required structure and implied behavior on how it works as a Vue component. When you see a `*.vue` file, you know it works differently from plain HTML.

- Vue templates are syntactically valid HTML, but the directives are essentially syntax extensions to express framework-specific intent.

- JSX has a spec, but isn't a standard. It's a non-standard syntax extension to JavaScript.

- TypeScript isn't a standard. It's a proprietary superset of JavaScript.

- Decorators has struggled to advance into the spec, yet is being widely used and Angular is completely built on top of it.

Granted, adding non-standard semantics to JavaScript still creates added learning cost and mental overhead, so we should carefully evaluate the trade-offs of each addition.

With that in mind, we believe `ref:`'s ergonomics value easily outweighs the cost. This is also why we are limiting this proposal to `ref:` only, since ref access is the only problem that requires alternative semantics to solve.

It can also be argued that the `ref:` syntax isn't far too removed from standard JavaScript. The following code is actually valid JavaScript in non-strict mode - if you copy it into an `.html` file, it will run as expected:

```html
<script>
ref: count = 0
console.log(count)
</script>
```

## Yet Another Way of Doing Things

Some may think that Vue already has Options API, Composition API, and Class API (outside of core, as a library) - and this RFC is adding yet another way of authoring a component. This is a valid concern, but it does not warrant an instant dismissal. When we talk about the drawbacks of "different ways of doing the same thing", the more fundamental issue is the learning cost incurred when a user encounters code written in another format he/she is not familiar with. It is therefore important to evaluate the addition based on the trade-off between:

- How much benefit does the new way provide?
- How much learning cost does the new way introduce?

This is what we did with the Composition API because we believed the scaling benefits provided by Composition API outweighs its learning cost.

Unlike the relatively significant paradigm difference between Options API and Composition API, this RFC is merely syntax sugar with the primary goal of reducing verbosity. It does not fundamentally alter the mental model. Without the ref sugar, Composition API code inside `<script setup>` will be 100% the same with normal Composition API usage (except for the lack of the return object, which is tedious and unnecessary in the first place). The `ref:` sugar is an extension of the ref concept: for a user with prior knowledge of Composition API, it shouldn't be difficult to quickly understand how it works.

That is to say - the syntax proposed in this RFC will not make the code more difficult to understand for someone who already knows Composition API. The initial time needed for a user to learn the syntax should be trivial compared to the improved DX in the long run.

## Requires dedicated tooling support

Appropriating the labeled statement syntax creates a semantic mismatch that leads to integration issues with tooling (linter, TypeScript, IDE support).

This was also one of the primary reservations we had about Svelte 3's design when it was initially proposed. However since then, the Svelte team has managed to provide good tooling/IDE support via its [language tools](https://github.com/sveltejs/language-tools), even for TypeScript.

Vue's single file component also already requires dedicated tooling like `eslint-plugin-vue` and Vetur. The team has already discussed the technical feasibility of providing such support and there should be no hard technical blocks to make it work. We are confident that we can provide:

- Special syntax highlight of `ref:` declared variables in Vetur (so that it's more obvious it's a reactive variable)
- Proper type check via Vetur and dedicated command line checker
- Proper linting via `eslint-plugin-vue`

## Extracting in-component logic

The `ref:` syntax sugar is only available inside single file components. Different syntax in and out of components makes it difficult to extract and reuse cross-component logic from existing components.

This is still an issue for Svelte, since Svelte compilation strategy only works inside Svelte components. The generated code assumes a component context and isn't human-maintainable.

In Vue's case, what we are proposing here is a very thin syntax sugar on top of idiomatic Composition API code. The most important thing to note here is that the code written with the sugar can be easily de-sugared into what a developer would have written without the sugar, and extracted into external JavaScript files for composition.

Given a piece of code written using the `ref:` sugar, the workflow of extracting it into an external composition function could be:

1. Select code range for the code to be extracted
2. In VSCode command input: `>vetur de-sugar ref usage'
3. Code gets de-sugared
4. Cut-paste code into external file and wrap into an exported function
5. Import the function in original file and replace original code.

# Alternatives

## Comment-based syntax

```html
<script setup>
import Foo from './Foo.vue'
import { computed } from 'vue'

// @ref
let count = 1

function inc() {
  count++
}
</script>

<template>
  <Foo :count="count" @click="inc" />
</template>
```

## Other related proposals

- https://github.com/vuejs/rfcs/pull/182 (current `<script setup>` implementation)
- https://github.com/vuejs/rfcs/pull/213
- https://github.com/vuejs/rfcs/pull/214

# Adoption strategy

This feature is opt-in. Existing SFC usage is unaffected.

# Unresolved questions

## Ref Usage in Nested Function Scopes

Technically, `ref:` doesn't have to be limited to root level scope and can be used anywhere `let` declarations can be used, including nested function scope:

```js
function useMouse() {
  ref: x = 0
  ref: y = 0

  function update(e) {
    x = e.pageX
    y = e.pageY
  }

  onMounted(() => window.addEventListener('mousemove', update))
  onUnmounted(() => window.removeEventListener('mousemove', update))

  return {
    x: $x,
    y: $y
  }
}
```

<details>
<summary>Compiled Output</summary>

```js
function useMouse() {
  const x = ref(0)
  const y = ref(0)

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

This will make the compilation (and accompanying linter / language service support) more complicated - I'm not sure if it's better to limit `ref:` usage to top scope bindings only.

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

## Template binding optimization

The `SFCScriptBlock` returned by `compiledScript` also exposes a `bindings` object, which is the exported binding metadata gathered during the compilation. For example, given the following `<script setup>`:

```vue
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
  foo: 'setup',
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

## Ref TypeScript Support Implementation Details

There are two issues that prevent `ref:` from working out of the box with TypeScript. Given the following code:

```ts
ref: count = x
```

1. TS won't know `count` should be treated as a local variable
2. If `x` has type `Ref<T>`, there will be a type mismatch since we expect to use `count` as `T`.

The general idea is to pre-transform the code into alternative TypeScript for type checking only (different from runtime-oriented output), get the diagnostics, and map them back. This will be performed by Vetur for IDE intellisense, and via a dedicated command line tool for type checking `*.vue` files (e.g. VTI or `@vuedx/typecheck`).

Example

```ts
// source
ref: count = x

// transformed
import { ref, unref } from 'vue'

let count = unref(x)
let $count = ref(x)
```

`ref` and `unref` here are used solely for type conversion purposes since their signatures are:

```ts
function ref<T>(value: T): T extends Ref ? T : Ref<T>
function unref<T>(value: T): T extends Ref<infer V> ? V : T
```

For destructuring:

```ts
// source
ref: ({ foo, bar } = useX())

// transformed
import { ref, unref } from 'vue'

const { foo: __foo, bar: __bar } = useX()
let foo = unref(__foo)
let $foo = ref(__foo)
let bar = unref(__bar)
let $bar = ref(__bar)
```

## Svelte Syntax Details

- `export` is used to created component props [[details](https://svelte.dev/docs#1_export_creates_a_component_prop)]

- `let` bindings are considered reactive (invalidation calls are automatically injected after assignments to `let` bindings during compilation). [[details](https://svelte.dev/docs#2_Assignments_are_reactive)]

- Labeled statements starting with `$` are used to denote computed values / reactive statements. [[details](https://svelte.dev/docs#3_$_marks_a_statement_as_reactive)]

- Imported svelte stores (the loose equivalent of a ref in Vue) can be used like a normal variable by using its `$`-prefixed counterpart. [[details](https://svelte.dev/docs#4_Prefix_stores_with_$_to_access_their_values)]
