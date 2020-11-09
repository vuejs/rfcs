- Start Date: 2020-10-28
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/rfcs/pull/182
- Implementation PR: https://github.com/vuejs/vue-next/pull/2532

# Summary

Introduce a compiler-based syntax sugar for using refs without `.value` inside `<script setup>` (as proposed in [#227](https://github.com/vuejs/rfcs/pull/227))

# Basic example

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

This proposal aims to improve the ergonomics of refs with the `ref:` syntax sugar.

Ever since the introduction of the Composition API, one of the primary unresolved questions is the use of refs vs. reactive objects. It can be cumbersome to use `.value` everywhere, and it is easy to miss if not using a type system. Some users specifically lean towards using `reactive()` exclusively so that they don't have to deal with refs.

The existence of ref is mostly a design trade-off due to the constraints of the language we are working with: JavaScript. JavaScript does not provide a native way to pass reactive bindings around without wrapping it with an object. This means that **it is impossible to use refs like normal variable bindings without altering or augmenting JavaScript semantics.**

- There has been a [proposal for adding native refs to JavaScript](https://github.com/rbuckton/proposal-refs), but it was designed to address a slightly different problem and doesn't seem to have received much attention.

- A prominent example of altering JavaScript semantics in return for succinct syntax is [Svelte](https://svelte/). It [appropriates a number of JavaScript syntax to express framework-specific behavior](#svelte-syntax-details).

In the past, we have tried to stick to strict JavaScript semantics as much as possible. Deviating from standard JavaScript semantics has number of [drawbacks](](#drawbacks)), but we believe there is room for a pragmatic trade-off where "breaking out of the box" a little bit can result in substantial improvements in developer experience.

# Detailed design

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

`ref: count = 0` is a [labeled statement](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/label) which is syntactically valid JavaScript (and hence TypeScript).

It is also semantically valid and executable in non-strict mode where assignment to a non-declared identifier implicitly declares a global variable. The code, however, will result in an error in strict mode where implicit global variable declarations are forbidden.

In this case, we are giving a piece of syntactically valid code different semantics. The compiler will:

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

# Drawbacks

## Non-standard semantics

> It has been pointed out that according to the [ES spec](https://tc39.es/ecma262/#sec-scripts-static-semantics-early-errors), duplicated labels with the same name violate static semantics and should result in a syntax error. This means a strict enough parser could refuse to accept it as valid source. However in practice, most major ES tooling including V8, Babel, TypeScript, ESLint and Prettier can parse (and even evaluate) duplicated labels just fine.

Some users may have strong aversion against non-standard semantics in their code, which is understandable. However, consider that:

- Single file components look like HTML but isn't actually HTML. It already has its own required structure and implied behavior on how it works as a Vue component. When you see a `*.vue` file, you know it works differently from plain HTML.

- Vue templates are syntactically valid HTML, but the directives are essentially syntax extensions to express framework-specific intent.

- JSX has a spec, but isn't a standard. It's a non-standard syntax extension to JavaScript.

- TypeScript isn't a standard. It's a proprietary superset of JavaScript.

- Decorators has struggled to advance into the spec, yet is being widely used and Angular is completely built on top of it.

Specifically for this RFC:

- The only syntax that is affected is the labeled statement syntax, which is a very rarely used syntax in practice.

- When they are actually used, labels are typically used to mark iteration statements (e.g. `for` or `while`), and paired with `continue` or `break`. In practice, **labeled assignment statements do not have any meaningful use cases with its original semantics.**

So, when we say "it breaks JavaScript semantics", we are talking about giving new semantics to a piece of syntax that is practically never used. Additionally, such syntax is only going to appear inside Vue files which is highly contextual. This means it is extremely unlikely to lead to confusions where users expect it to carry the original semantics.

With that in mind, we believe `ref:`'s ergonomics value outweighs the cost by a fair margin. This is also why we are limiting this proposal to `ref:` only, since ref access is the only problem that requires alternative semantics to solve.

## Requires dedicated tooling support

Appropriating the labeled statement syntax creates a semantic mismatch that leads to integration issues with tooling (linter, TypeScript, IDE support).

This was also one of the primary reservations we had about Svelte 3's design when it was initially proposed. However since then, the Svelte team has managed to provide good tooling/IDE support via its [language tools](https://github.com/sveltejs/language-tools), even for TypeScript.

Vue's single file component also already requires dedicated tooling like `eslint-plugin-vue` and Vetur. The team has already discussed the technical feasibility of providing such support and there should be no hard technical blocks to make it work. We are confident that we can provide:

- Special syntax highlight of `ref:` declared variables in Vetur (so that it's more obvious it's a reactive variable)
- Proper type check via Vetur and dedicated command line checker
- Proper linting via `eslint-plugin-vue`

## Different Mental Models in/out of Components

The `ref:` syntax sugar is only available inside single file components. Different syntax in and out of components creates a mental model shift cost. [This study](https://github.com/vuejs/rfcs/pull/222#issuecomment-723560606) shows that this mental cost may actually reduce efficiency compared to usage without the sugar.

Differnet syntax also makes it difficult to extract and reuse cross-component logic from existing components.

This is still an issue for Svelte, since Svelte compilation strategy only works inside Svelte components. The generated code assumes a component context and isn't human-maintainable.

In Vue's case, it should be noted that the code written with the sugar can be easily de-sugared into what a developer would have written without the sugar, and extracted into external JavaScript files for composition.

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
