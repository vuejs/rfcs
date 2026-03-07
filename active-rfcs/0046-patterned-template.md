- Start Date: 2026-03-05
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: (leave this empty)

# Summary

Introduce a pattern matching directive for Vue templates that enables declarative, multi-branch conditional rendering based on the value of an expression. This provides a cleaner alternative to chained `v-if` / `v-else-if` / `v-else` when branching on a single expression's value.

# Basic example

```html
<template>
  <div v-match="status">
    <p v-case="'loading'">Loading...</p>
    <p v-case="'error'">Something went wrong.</p>
    <p v-case="'success'">Data loaded successfully!</p>
    <p v-case.default>Unknown status.</p>
  </div>
</template>
```

This is equivalent to:

```html
<template>
  <p v-if="status === 'loading'">Loading...</p>
  <p v-else-if="status === 'error'">Something went wrong.</p>
  <p v-else-if="status === 'success'">Data loaded successfully!</p>
  <p v-else>Unknown status.</p>
</template>
```

# Motivation

## The problem with chained `v-if` / `v-else-if`

When rendering different content based on the value of a single reactive expression, developers must repeatedly reference the same expression in a chain of `v-if` / `v-else-if` directives:

```html
<template>
  <div v-if="result.tag === 'ok'">{{ result.value }}</div>
  <div v-else-if="result.tag === 'err'">{{ result.error }}</div>
  <div v-else-if="result.tag === 'pending'">Loading...</div>
  <div v-else>Unknown</div>
</template>
```

This pattern has several drawbacks:

1. **Repetition**: The discriminant expression (`result.tag`) is repeated in every branch, violating DRY and increasing the chance of typos.
2. **Readability**: It is hard to see at a glance that all branches are testing the same expression.
3. **Fragility**: Adding or reordering branches requires careful attention to the `v-else-if` chain.
4. **Lack of intent**: The `v-if` chain doesn't communicate that this is exhaustive matching on a single value — it looks the same as unrelated conditions.

## Growing need for discriminated unions in Vue

TypeScript's discriminated unions and tagged union patterns are widely adopted in Vue 3 codebases. Libraries like VueQuery, Pinia, and vue-router commonly return status objects (e.g., `{ status: 'loading' | 'error' | 'success', ... }`). A dedicated pattern matching directive makes it natural to consume these patterns in templates.

## Prior art

- **JavaScript TC39 Pattern Matching Proposal** (Stage 1): `match (expr) { when ... }`
- **Svelte**: `{#if}` / `{:else if}` chains (no dedicated switch)
- **Rust**: `match` expression
- **Swift**: `switch` statement with pattern matching
- **Kotlin**: `when` expression

# Detailed design

## Syntax candidates

We propose several syntax candidates. The naming of the directive pair is a key design decision.

### Option A: `v-match` / `v-case` (Recommended)

```html
<div v-match="expression">
  <p v-case="'value1'">...</p>
  <p v-case="'value2'">...</p>
  <p v-case.default>...</p>
</div>
```

- `v-match` communicates "pattern matching" intent, aligning with TC39 proposal and Rust naming.
- `v-case` is familiar from `switch/case` in JavaScript.
- `.default` modifier for the fallback branch.

### Option B: `v-switch` / `v-case`

```html
<div v-switch="expression">
  <p v-case="'value1'">...</p>
  <p v-case="'value2'">...</p>
  <p v-case.default>...</p>
</div>
```

- Most familiar to JavaScript developers (`switch/case`).
- However, `v-switch` may imply fall-through semantics, which this directive does **not** have.

### Option C: `v-match` / `v-when`

```html
<div v-match="expression">
  <p v-when="'value1'">...</p>
  <p v-when="'value2'">...</p>
  <p v-when.default>...</p>
</div>
```

- Aligns with TC39 `match/when` and Kotlin's `when`.
- Reads naturally: "match expression — when value1, when value2."

### Option D: `v-pattern` / `v-case`

```html
<div v-pattern="expression">
  <p v-case="'value1'">...</p>
  <p v-case="'value2'">...</p>
  <p v-case.default>...</p>
</div>
```

- Explicit about the "pattern matching" concept.
- Slightly more verbose.

### Recommendation

**Option A (`v-match` / `v-case`)** is the recommended syntax because:

- `match` is concise and well-recognized across languages as a pattern matching keyword.
- `case` is universally understood as a branch within a match/switch.
- No fall-through ambiguity (unlike `v-switch`).
- Aligns with the likely direction of the TC39 pattern matching proposal.

## Semantics

### Basic matching (strict equality)

`v-match` evaluates its expression once per render. Each `v-case` child is compared against the result using strict equality (`===`). Only the first matching `v-case` branch is rendered.

```html
<template v-match="theme">
  <link v-case="'dark'" rel="stylesheet" href="/dark.css" />
  <link v-case="'light'" rel="stylesheet" href="/light.css" />
  <link v-case.default rel="stylesheet" href="/default.css" />
</template>
```

### Using `<template>` as the `v-match` host

Like `v-if`, `v-match` can be placed on a `<template>` element to avoid introducing an extra wrapper DOM node:

```html
<template v-match="user.role">
  <template v-case="'admin'">
    <AdminDashboard />
    <AdminTools />
  </template>
  <template v-case="'editor'">
    <EditorDashboard />
  </template>
  <template v-case.default>
    <ViewerDashboard />
  </template>
</template>
```

### Multiple values per case

A single `v-case` can match against multiple values using an array:

```html
<template v-match="httpStatus">
  <p v-case="[200, 201, 204]">Success</p>
  <p v-case="[301, 302]">Redirect</p>
  <p v-case="[400, 401, 403]">Client Error</p>
  <p v-case="[500, 502, 503]">Server Error</p>
  <p v-case.default>Unknown Status</p>
</template>
```

When `v-case` receives an array, the branch matches if the discriminant is strictly equal to **any** element in the array (i.e., `array.includes(discriminant)`).

### Default branch

The `.default` modifier designates a fallback branch. It is rendered when no other `v-case` matches. At most one `v-case.default` is allowed per `v-match` block. If no `.default` is provided and no case matches, nothing is rendered (same behavior as `v-if` with no `v-else`).

```html
<div v-match="answer">
  <p v-case="42">Correct!</p>
  <p v-case.default>Try again.</p>
</div>
```

The value expression on `v-case.default` is optional and ignored. Writing `v-case.default` without a value is the idiomatic style.

### Nested `v-match`

`v-match` blocks can be nested:

```html
<template v-match="response.status">
  <template v-case="'success'">
    <template v-match="response.data.type">
      <UserCard v-case="'user'" :data="response.data" />
      <PostCard v-case="'post'" :data="response.data" />
    </template>
  </template>
  <template v-case="'error'">
    <ErrorMessage :error="response.error" />
  </template>
</template>
```

### Reactivity

`v-match` is fully reactive. When the discriminant expression changes, Vue re-evaluates which branch to render and patches the DOM accordingly. Each `v-case` expression is also reactive — if a `v-case` value is a ref or computed, changes to it will trigger re-evaluation.

### Compilation

The compiler transforms `v-match` / `v-case` into an equivalent `v-if` / `v-else-if` / `v-else` chain internally. For example:

```html
<div v-match="status">
  <p v-case="'a'">A</p>
  <p v-case="'b'">B</p>
  <p v-case.default>Default</p>
</div>
```

Compiles to the equivalent of:

```js
// Pseudocode for the generated render function
const __match_0 = status

if (__match_0 === 'a') {
  // render <p>A</p>
} else if (__match_0 === 'b') {
  // render <p>B</p>
} else {
  // render <p>Default</p>
}
```

The discriminant is evaluated once and cached in a temporary variable to avoid redundant evaluations.

For multiple values:

```js
const __match_0 = httpStatus

if ([200, 201, 204].includes(__match_0)) {
  // render Success
} else if ([301, 302].includes(__match_0)) {
  // render Redirect
} ...
```

## Validation rules

The compiler should enforce the following rules and emit warnings/errors:

1. **`v-case` must be a direct child of a `v-match` element.** Using `v-case` outside of `v-match` is a compile-time error.
2. **At most one `v-case.default` per `v-match`.** Multiple defaults produce a compile-time error.
3. **`v-case` must not be combined with `v-if`, `v-else-if`, `v-else`, `v-for`, or another `v-match` on the same element.** These are mutually exclusive structural directives.
4. **Non-`v-case` children of `v-match` are ignored with a warning.** All direct children of a `v-match` element should use `v-case`.
5. **`v-match` with no `v-case` children** produces a compile-time warning.

## TypeScript support

When using `v-match` on a discriminated union, Volar (vue-tsc) should narrow the type within each `v-case` branch:

```vue
<script setup lang="ts">
type Result =
  | { tag: 'ok'; value: string }
  | { tag: 'err'; error: Error }

const result = ref<Result>({ tag: 'ok', value: 'hello' })
</script>

<template>
  <template v-match="result.tag">
    <!-- result is narrowed to { tag: 'ok'; value: string } here -->
    <p v-case="'ok'">{{ result.value }}</p>
    <!-- result is narrowed to { tag: 'err'; error: Error } here -->
    <p v-case="'err'">{{ result.error.message }}</p>
  </template>
</template>
```

This requires Volar plugin support and is a stretch goal for the initial implementation.

# Drawbacks

- **New directive pair**: Adds two new built-in directives (`v-match` and `v-case`) to Vue's API surface. This increases the learning surface, though both concepts are widely understood.
- **Overlap with `v-if` / `v-else-if`**: This feature doesn't enable anything new — it's syntactic sugar over existing directives. Developers need to learn when to use which.
- **Compiler complexity**: The compiler must handle the parent-child relationship between `v-match` and `v-case`, which is a new pattern for Vue structural directives (currently all structural directives are independent).
- **Userland alternative**: This can be approximated in userland with a renderless component, though with worse ergonomics and no compiler optimizations.

# Alternatives

## 1. Do nothing — keep using `v-if` / `v-else-if`

The current approach works and is well-understood. The cost is verbosity and repeated expressions.

## 2. Renderless `<Match>` component

```html
<Match :value="status">
  <Case value="loading"><p>Loading...</p></Case>
  <Case value="error"><p>Error!</p></Case>
  <Case :default><p>Unknown</p></Case>
</Match>
```

This can be built in userland but suffers from:
- Extra runtime overhead (component instances for each case)
- Cannot leverage compiler optimizations
- Awkward slot scoping for type narrowing

## 3. Expression-based approach (no wrapper)

```html
<p v-match:status="'loading'">Loading...</p>
<p v-match:status="'error'">Error!</p>
<p v-match:status.default>Unknown</p>
```

Using the directive argument as the discriminant avoids a wrapper element but:
- Requires repeating the discriminant name on every branch
- Loses the visual grouping of related branches
- Has ambiguous scoping if used non-consecutively

## 4. `v-switch` / `v-case` naming

As discussed in the syntax candidates section, `v-switch` is more familiar but implies fall-through behavior. We recommend `v-match` to avoid this confusion.

# Adoption strategy

- This is a **non-breaking, additive** feature. No existing code needs to change.
- `v-if` / `v-else-if` / `v-else` remains fully supported and is still appropriate for conditions that test different expressions.
- ESLint plugin (`eslint-plugin-vue`) can provide a rule to suggest `v-match` when it detects a `v-if` / `v-else-if` chain that tests the same expression.
- Documentation should present `v-match` alongside `v-if` in the conditional rendering guide, with guidance on when to use each.
- A codemod can be provided to automatically convert eligible `v-if` chains to `v-match` / `v-case`.

# Unresolved questions

1. **Naming**: Should we go with `v-match` / `v-case`, `v-switch` / `v-case`, `v-match` / `v-when`, or something else? This RFC presents the trade-offs but the final decision should be made through community discussion.
2. **Guard conditions**: Should `v-case` support an additional guard expression (e.g., `v-case="'error'" v-case-if="retryCount < 3"`)? This adds power but also complexity. It could be deferred to a follow-up RFC.
3. **Exhaustiveness checking**: Should the compiler or Volar warn when a `v-match` on a union type is missing branches? This is valuable for type safety but requires deep integration with the type system.
4. **Range matching**: Should there be syntax for matching ranges (e.g., `v-case="1..10"`)? This is useful but may be better handled by `v-if` or deferred.
5. **Deep pattern matching**: Should future iterations support matching on object shape (e.g., `v-case="{ status: 'ok', data: { type: 'user' } }"`)? This is powerful but complex and should likely be a separate RFC.
6. **`v-case` value expression evaluation**: Should `v-case` values be limited to literals for optimization, or should any expression be allowed? Allowing expressions provides flexibility but prevents some compile-time optimizations.
