- Start Date: 2026-03-05
- Target Major Version: ?
- Reference Issues: N/A
- Implementation PR: (leave this empty)

# Summary

Introduce `v-match` and `v-when` for declarative pattern-based conditional rendering in Vue templates.

`v-match` evaluates a subject expression once. Each direct `v-when` child declares a pattern, optional bindings (including object and array rest), and an optional guard. Branches are checked from top to bottom, only the first matching branch renders, and no fallthrough is possible. Template type checking requires exhaustive coverage, with `_` available as the fallback pattern.

The initial syntax keeps `v-when` in its long form. Shorthand candidates and their trade-offs are recorded below for future discussion.

The proposed syntax intentionally follows the vocabulary of the TC39 ECMAScript Pattern Matching proposal (`match` / `when`, pattern guards, declaration bindings) while also catching up with Flow's shipped `match` feature (`const` variable declaration patterns, `if` guards, `_` wildcard, `|` alternatives, `as` bindings, and exhaustiveness-aware tooling).

# Basic example

```vue
<script setup lang="ts">
type Result =
  | { status: 'loading' }
  | { status: 'success'; data: Article }
  | { status: 'error'; error: FetchError }

const result = ref<Result>({ status: 'loading' })
</script>

<template>
  <template v-match="result">
    <ArticleView
      v-when="{ status: 'success', data: const article }"
      :article="article"
    />

    <RetryBanner
      v-when="{ status: 'error', error: const error } if (error.retriable)"
      :error="error"
    />

    <ErrorBanner
      v-when="{ status: 'error', error: const error }"
      :error="error"
    />

    <LoadingSpinner v-when="{ status: 'loading' }" />
    <p v-when="_">Unknown result.</p>
  </template>
</template>
```

The above is equivalent in behavior to a `v-if` / `v-else-if` chain that first evaluates `result`, checks each branch in order, introduces branch-local template bindings such as `article` and `error`, and renders the fallback only if no previous branch matched.

# Motivation

## Chained `v-if` is repetitive for one subject

When rendering different content based on a single reactive value, developers currently repeat the same expression in every branch:

```vue
<template>
  <ArticleView v-if="result.status === 'success'" :article="result.data" />
  <RetryBanner
    v-else-if="result.status === 'error' && result.error.retriable"
    :error="result.error"
  />
  <ErrorBanner v-else-if="result.status === 'error'" :error="result.error" />
  <LoadingSpinner v-else-if="result.status === 'loading'" />
  <p v-else>Unknown result.</p>
</template>
```

This has several drawbacks:

1. The discriminant expression is repeated in every branch.
2. Branches that conceptually belong to one match are only implicitly grouped by adjacency.
3. Nested data has to be re-addressed manually instead of being bound where it is matched.
4. Type tooling has to recover intent from arbitrary boolean expressions.

`v-match` makes the subject explicit, and `v-when` makes each branch an arm of the same match.

## Vue code increasingly models UI state as tagged data

Vue applications commonly consume typed state from composables, loaders, data-fetching libraries, state stores, routers, and RPC clients. These values are often modeled as discriminated unions or tagged objects:

```ts
type RemoteData<T, E> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: E }
```

Templates need an ergonomic way to render these states without repeatedly indexing into the same object and without losing branch-specific type information.

## Prior art is converging on patterns plus bindings

- The [TC39 ECMAScript Pattern Matching proposal](https://github.com/tc39/proposal-pattern-matching) uses `match` expressions with `when` arms, declaration patterns such as `const status`, rest binding patterns, and guard patterns.
- [Flow's `match`](https://flow.org/en/docs/match/) has shipped with object, array, wildcard, `const`, `|`, `as`, instance, guard, exhaustiveness, and unused-pattern checks.
- Rust, Swift, Kotlin, Scala, F#, Python, and Elixir all demonstrate that pattern matching is most valuable when it can both test a shape and bind useful parts of the matched value.

This RFC does not attempt to add JavaScript pattern matching to Vue. It proposes a template-level feature that borrows the parts that map cleanly to rendering: arm syntax, structural patterns, branch-local bindings, guard conditions, and type-tooling hooks.

# Detailed design

## Recommended syntax: `v-match` / `v-when`

```html
<template v-match="expression">
  <template v-when="<pattern>">...</template>
  <template v-when="<pattern> if (<guard>)">...</template>
  <template v-when="_">...</template>
</template>
```

- `v-match` evaluates the subject expression.
- `v-when` declares a pattern arm.
- `v-when="_"` declares an unconditional fallback arm.
- An optional `if (<guard>)` suffix acts as a branch guard.

The name `v-when` is recommended over the previous `v-case` direction because it directly mirrors TC39's `match (...) { when ... }` vocabulary and avoids suggesting JavaScript `switch` fallthrough behavior.

## Shorthand discussion (deferred)

Vue already provides short forms such as [`@` for `v-on`](https://vuejs.org/api/built-in-directives.html#v-on), [`:` for `v-bind`](https://vuejs.org/api/built-in-directives.html#v-bind), and [`#` for `v-slot`](https://vuejs.org/api/built-in-directives.html#v-slot). A short form of `v-when` could make repeated arms similarly concise. The candidates below have readability drawbacks, so this RFC defers a shorthand and recommends the long form.

### `?`: conditional branching

`?` takes its cue from JavaScript's conditional operator, `condition ? consequent : alternate`. It suggests a branch selected by a condition and avoids spelling a JavaScript compound assignment when followed by the attribute's `=`:

```html
<template v-match="result">
  <ArticleView ?="{ status: 'success', const data }" :article="data" />
  <ErrorBanner ?="{ status: 'error', const error }" :error="error" />
  <p ?="_">Unknown result.</p>
</template>
```

Its weakness is that `?` suggests conditional rendering generally, so it could be mistaken for a `v-if` shorthand. Its value would still be a pattern, not a truthiness test: `?="true"` would match the subject against `true`. This mnemonic is proposed for Vue; it is not shorthand defined by TC39 or Flow.

### `|`: pattern-matching branches

`|` has a direct precedent in [OCaml pattern matching](https://ocaml.org/docs/basic-data-types#lists) and [F# match expressions](https://learn.microsoft.com/en-us/dotnet/fsharp/language-reference/match-expressions), where `| pattern -> expression` introduces an alternative branch.

However, writing it as an attribute produces `|="pattern"`, which visually reads as JavaScript's bitwise OR assignment operator `|=`. Combining it with or-patterns further repeats the symbol:

```html
<p |="'idle' | 'loading'">Waiting...</p>
```

The attribute name and quoted pattern are unambiguous to a parser, but the compound-assignment resemblance is a readability drawback. `|` is therefore not recommended as the attribute shorthand. This does not affect `|` inside patterns, which retains its role as the or combinator.

### `~` or no shorthand

`~="pattern"` can suggest similarity or matching, but that is a weaker mnemonic for selecting a branch, and JavaScript uses `~` for bitwise negation. Keeping only `v-when` remains a viable choice if none of the symbols reads clearly enough.

`v-match` remains the explicit name of the enclosing construct. It appears once per block, while `v-when` is repeated for every arm, so shortening the arms provides most of the benefit and keeps the subject easy to find. A second symbol for the host is not proposed.

### Parsing and tooling

Any selected shorthand must normalize to `v-when` before structural validation, binding analysis, and code generation. Long and short forms must produce the same directive AST apart from source locations, preserve pattern bindings and guards, and never emit the shorthand as an HTML attribute or component prop, including during SSR. Both forms require a direct `v-match` parent and can be mixed across arms.

The pattern, including the `_` fallback, stays in the quoted attribute value. Neither form accepts directive arguments or modifiers; a missing pattern and combining long and short forms on the same element are compile-time errors. The guard after `if` remains a normal JavaScript expression, including its usual operators. Existing `@`, `:`, and `#` syntax keeps its meaning inside `v-match`.

Editor highlighting, completion, formatting, linting, and Volar / vue-tsc would need to recognize both forms and preserve the source spelling. These candidates are not implemented by the current Vue parser. Shorthand can be deferred independently of the long form.

## Branch order

Branches are tested in source order. Only the first branch whose pattern and guard match is rendered.

```html
<template v-match="status">
  <p v-when="'loading' | 'pending'">Loading...</p>
  <p v-when="'error'">Something went wrong.</p>
  <p v-when="_">Done.</p>
</template>
```

The above renders the first branch when `status` is either `'loading'` or `'pending'`, the second when it is `'error'`, and the wildcard branch otherwise.

## Pattern grammar

`v-when` uses a pattern grammar, not a normal JavaScript expression grammar. This is similar to how `v-for` already has directive-specific syntax.

The initial pattern grammar should include the subset that is most useful in templates and that aligns with TC39 and Flow:

```txt
Pattern:
  LiteralPattern
  ValuePattern
  WildcardPattern
  BindingPattern
  ObjectPattern
  ArrayPattern
  OrPattern
  AsPattern
  ParenthesizedPattern
```

### Literal patterns

Literal patterns match primitive values:

```html
<p v-when="'success'">Done</p>
<p v-when="404">Not found</p>
<p v-when="true">Enabled</p>
<p v-when="null">No value</p>
```

Literal matching uses JavaScript strict equality semantics, with `NaN` matched using `Number.isNaN`.

### Value patterns

Identifiers and member expressions can be used as value patterns:

```html
<template v-match="status">
  <p v-when="Status.Active">Active</p>
  <p v-when="fallbackStatus">Fallback</p>
</template>
```

A value pattern compares the subject to the runtime value of the expression. To keep the initial feature deterministic and friendly to tooling, arbitrary expressions are not allowed as patterns. Developers can assign an expression to a binding in `<script setup>` and match against that binding.

### Wildcard pattern

`_` matches any remaining value:

```html
<template v-match="tab">
  <HomePanel v-when="'home'" />
  <SettingsPanel v-when="'settings'" />
  <NotFoundPanel v-when="_" />
</template>
```

`v-when="_"` is the idiomatic fallback. `_` also matches any value at a nested pattern position, such as `{ status: _ }`, without introducing a binding. To access the matched value, use a binding pattern instead.

### Binding patterns

A binding pattern matches any value and exposes it as a branch-local template binding:

```html
<template v-match="message">
  <p v-when="const text">{{ text }}</p>
</template>
```

For the initial RFC, `const` is the only supported binding declaration:

- It is a compatible subset of TC39 declaration patterns.
- It matches Flow's currently type-supported variable declaration patterns.
- It matches the template mental model better than mutable `let` or function-scoped `var`.

`let` and `var` are reserved for future discussion and should produce compile-time errors in the initial implementation.

Bindings are scoped to the matching branch only. They are visible to the element that carries `v-when`, its other directive expressions, its attributes, its children, and the optional guard expression. They are not visible to sibling branches or outside the `v-match` block.

### Object patterns

Object patterns match object shape and can introduce bindings:

```html
<template v-match="result">
  <ArticleView
    v-when="{ status: 'success', data: const article }"
    :article="article"
  />

  <ErrorBanner
    v-when="{ status: 'error', error: const error }"
    :error="error"
  />
</template>
```

Shorthand binding follows Flow's object pattern spelling:

```html
<template v-match="result">
  <ArticleView v-when="{ status: 'success', const data }" :article="data" />
</template>
```

`{ const data }` is equivalent to `{ data: const data }`.

Object rest can bind the remaining own enumerable properties:

```html
<template v-match="result">
  <ErrorDetails
    v-when="{ status: 'error', ...const payload }"
    :payload="payload"
  />
</template>
```

Runtime matching follows open structural matching: extra properties do not make the pattern fail. Exhaustiveness analysis must use the same rule; it must not treat omission of `...` as a demand for an exact set of properties.

A lone `...` is allowed to explicitly acknowledge additional properties, borrowing Flow's spelling. Under this RFC's open object semantics it does not change runtime matching or coverage and introduces no binding:

```html
<ErrorPanel v-when="{ status: 'error', ... } as failure" :result="failure" />
```

### Array patterns

Array patterns match arrays and tuples structurally. They require `Array.isArray(subject)`. Without rest, the subject must have exactly the listed number of elements; a trailing `...` or `...const rest` permits additional elements. An array rest binding collects those additional elements. General iterables and array-like objects are not included in the initial template grammar.

```html
<template v-match="color">
  <span v-when="[0, 0, 0]">black</span>
  <span v-when="[255, 0, 0, ...]">red</span>
  <span v-when="[0, 0, 255, ...const rest]"> blue alpha: {{ rest[0] }} </span>
</template>
```

This is a change from the earlier draft where an array in `v-case` meant "one of these values". In the revised design, arrays are structural patterns, matching TC39 and Flow. Use `|` for multiple alternatives.

### Rest patterns and bindings

Object and array rest are required in the initial feature, including runtime support, branch-local type inference, and exhaustiveness analysis. They use the following syntax inside an object or array pattern:

```txt
RestPattern:
  ...
  ...const BindingIdentifier
```

`...` accepts the remainder without binding it. `...const rest` accepts the same values and binds the remainder. These forms borrow the pattern-specific rest spelling from TC39 and Flow; `...rest` without `const` is not supported. Rest is not a standalone top-level pattern.

For object rest, every explicitly listed key is excluded from the result, whether its subpattern is a literal, wildcard, or binding. The rest binding is a fresh ordinary object containing the remaining own enumerable string and symbol properties, with JavaScript object-rest copy semantics. Inherited and non-enumerable properties are not copied. An empty remainder is valid and binds `{}`.

```html
<template v-match="message">
  <MessageCard
    v-when="{ kind: 'message', const text, ...const metadata }"
    :text="text"
    v-bind="metadata"
  />
  <template v-when="_"></template>
</template>
```

For `{ kind: 'message', text: 'Hello', sender: 'Ada' }`, `metadata` is `{ sender: 'Ada' }`; both `kind` and `text` are excluded.

For array rest, the listed prefix must match first. The rest binding is a fresh array containing the remaining element values in index order, without copying non-index properties. Rest accepts zero additional elements: `[const first, ...const tail]` matches any non-empty array, and matching `[42]` binds `first` to `42` and `tail` to `[]`. `[...const items]` matches any array, including an empty one.

```html
<template v-match="items">
  <EmptyState v-when="[]" />
  <ItemList
    v-when="[const first, ...const remaining]"
    :first="first"
    :remaining="remaining"
  />
</template>
```

For an array-typed subject this match is exhaustive: `[]` covers length zero, and the second arm covers every positive length. `[]` and `[const first]` alone would leave arrays with two or more elements uncovered. A bare `[ ... ]` is likewise exhaustive for an array-typed subject, but not for `unknown` or an array-or-`null` union.

Rest can appear at multiple nesting levels, with at most one rest entry per object or array pattern:

```html
<ResultList
  v-when="{ items: [const first, ...const remaining], ...const metadata }"
  :first="first"
  :remaining="remaining"
  :metadata="metadata"
/>
```

Rest bindings have the same scope as other pattern bindings and are available in the arm's guard. They are initialized when their pattern succeeds, before that guard runs. A failed guard discards the arm's bindings and continues matching. Copies are shallow: nested objects retain their identity, the source is not mutated, and the binding's `const` does not deep-freeze the copied value. Copying participates in normal reactive reads; rest values may be recreated when rendering is reevaluated, with no stable-identity guarantee. Unbound `...` requires no remainder allocation or reads of discarded values.

The checker infers object rest using TypeScript's object-rest rules on the narrowed subject, excluding the listed keys for each remaining union member. For tuple rest it preserves the known tail: matching `[const first, ...const tail]` against `[string, number, boolean]` gives `tail` the type `[number, boolean]`. For a general `string[]`, `tail` is `string[]`. Both rest forms copy into new containers, including when the input container is readonly.

Rest must be last in its enclosing pattern and must not have a trailing comma. Multiple rest entries, elements or properties after rest, `...let rest`, `...var rest`, and duplicate binding names are compile-time errors. Rest bindings inside an or-pattern follow the initial restriction on bindings in alternatives. Nested destructuring directly after `...` is not supported; authors can match the bound remainder in a nested `v-match`.

### Or patterns

`|` combines multiple patterns:

```html
<template v-match="status">
  <p v-when="'idle' | 'loading'">Waiting...</p>
  <p v-when="'success'">Done.</p>
  <p v-when="'error'">Failed.</p>
</template>
```

TC39 currently spells this combinator as `or`, while Flow uses `|`. This RFC recommends `|` for Vue templates because it matches Flow's shipped syntax and the union-like notation TypeScript users already read in type positions. Supporting `or` as a future alias remains possible.

For the initial implementation, bindings inside `|` patterns are not supported. This follows Flow's current restriction and keeps branch-local binding types predictable:

```html
<!-- Compile-time error in the initial proposal -->
<p
  v-when="{ status: 'success', data: const value } | { status: 'error', error: const value }"
>
  {{ value }}
</p>
```

Authors can use separate arms instead.

### As patterns

`as` patterns bind the whole matched value after the pattern succeeds:

```html
<template v-match="result">
  <ErrorPanel v-when="{ status: 'error', ... } as failure" :result="failure" />
</template>
```

This mirrors Flow's `as` pattern and gives templates a concise way to pass the refined object itself while still testing its shape.

### Parenthesized patterns

Parentheses can disambiguate complex patterns:

```html
<p v-when="('idle' | 'loading') as pendingStatus">{{ pendingStatus }}</p>
```

## Guards

A `v-when` pattern can be followed by an `if (<guard>)` suffix:

```html
<template v-match="result">
  <RetryBanner
    v-when="{ status: 'error', error: const error } if (error.retriable)"
    :error="error"
  />
  <ErrorBanner
    v-when="{ status: 'error', error: const error }"
    :error="error"
  />
</template>
```

The guard runs only after the pattern succeeds. Pattern bindings are available inside the guard.

This spelling follows Flow's guard placement. It also corresponds to TC39 guard patterns: `when <pattern> and if (<guard>)`. Vue should document the simpler template spelling while keeping the conceptual mapping clear.

Guarded arms do not contribute to exhaustive coverage, even if a guard appears constant. A guard can fail at runtime even when the structural pattern matched; an unguarded arm must cover the remaining values. The initial checker does not attempt to prove relationships between guard expressions.

## Wildcard fallback

An unguarded `_` arm provides the fallback using the same pattern grammar as every other branch:

```html
<template v-match="answer">
  <p v-when="42">Correct</p>
  <p v-when="_">Try again</p>
</template>
```

Rules:

1. An unguarded top-level `_` arm must be last and unique within its `v-match` block, including when parenthesized.
2. A guarded wildcard such as `_ if (showFallback)` is an ordinary conditional arm. If its guard fails, matching continues; it does not count as exhaustive coverage or as the unconditional fallback.
3. Nested wildcards such as `{ status: _ }` only accept values at that position. The enclosing structural pattern must still match.
4. The fallback is optional when other arms cover the subject type. Type tooling reports incomplete coverage as specified below. At runtime, if no arm matches, nothing is rendered, as with `v-if` without `v-else`; static exhaustiveness does not introduce a runtime exception.
5. No dedicated `.default` modifier is proposed. A fallback that needs the subject can use `const value` or `_ as value` instead of a bare wildcard.

For example, a guarded wildcard can precede an unconditional one:

```html
<template v-match="result">
  <ArticleView v-when="{ status: 'success', const data }" :article="data" />
  <DebugPanel v-when="_ if (debug)" :result="result" />
  <p v-when="_">No result.</p>
</template>
```

## Reactivity and evaluation

`v-match` evaluates its subject expression once per render and stores it in a compiler-generated temporary.

Each branch pattern is checked against that temporary. Branch-local bindings are derived from the matched subject and participate in normal template reactivity because they are re-created on each render from reactive source values.

```html
<template v-match="expensiveResult()">
  <ArticleView
    v-when="{ status: 'success', data: const article }"
    :article="article"
  />
  <p v-when="_">Nothing to render.</p>
</template>
```

The compiler must not call `expensiveResult()` once per branch.

## Compilation

The compiler can lower `v-match` to an equivalent conditional chain.

For:

```html
<template v-match="result">
  <ArticleView
    v-when="{ status: 'success', data: const article }"
    :article="article"
  />
  <ErrorBanner
    v-when="{ status: 'error', error: const error } if (error.retriable)"
    :error="error"
  />
  <p v-when="_">Unknown</p>
</template>
```

The generated render logic is conceptually:

```js
const __match_0 = result

match_0: {
  if (
    __match_0 != null &&
    __match_0.status === 'success' &&
    'data' in Object(__match_0)
  ) {
    const article = __match_0.data
    // render <ArticleView :article="article" />
    break match_0
  }

  if (
    __match_0 != null &&
    __match_0.status === 'error' &&
    'error' in Object(__match_0)
  ) {
    const error = __match_0.error
    if (error.retriable) {
      // render <ErrorBanner :error="error" />
      break match_0
    }
  }

  // render <p>Unknown</p>
}
```

The actual implementation can use compiler IR helpers instead of literally emitting this structure. The important properties are:

1. The subject is evaluated once.
2. Branches are checked in source order.
3. Bindings are initialized only for the branch that matched.
4. Guards run after pattern bindings are initialized.
5. A failed guard continues matching later branches.

## Validation rules

The compiler should enforce these rules, also applying them to any shorthand adopted later:

1. `v-when` must be a direct child of a `v-match` element or `<template>`.
2. `v-match` with no `v-when` children should warn.
3. Non-`v-when` direct children of a `v-match` block should warn.
4. An unguarded top-level `_` arm must be last and unique, including across long and short forms.
5. Every `v-when` must have a valid pattern and must not have arguments or modifiers.
6. `v-when` must not be combined with `v-if`, `v-else-if`, `v-else`, `v-for`, or another `v-match` on the same element.
7. Binding names must be valid identifiers and must not collide within a single pattern.
8. `let` and `var` binding patterns should be compile-time errors in the initial implementation.
9. Bindings inside `|` patterns should be compile-time errors in the initial implementation.
10. Unreachable branches should warn when the compiler or type tooling can prove a previous branch already covers them.
11. An element must not declare multiple arms, including through a combination of long and short forms if a shorthand is adopted.
12. Each object or array pattern may have at most one rest entry, in its final position with no trailing comma. A bound rest must use `...const identifier` and obey the same binding-name and or-pattern restrictions as other bindings.

## Type tooling

Volar / vue-tsc must use `v-match` and `v-when` as type-narrowing boundaries.

```vue
<script setup lang="ts">
type Result =
  | { status: 'success'; data: Article }
  | { status: 'error'; error: Error }

defineProps<{ result: Result }>()
</script>

<template>
  <template v-match="result">
    <ArticleView
      v-when="{ status: 'success', data: const article }"
      :article="article"
    />
    <ErrorBanner
      v-when="{ status: 'error', error: const error }"
      :error="error"
    />
  </template>
</template>
```

Inside the first branch, `result` is narrowed to `{ status: 'success'; data: Article }`, and `article` is typed as `Article`. Inside the second branch, `error` is typed as `Error`. `as` bindings preserve the narrowed whole-subject type. A nested `v-match` starts from the type available within its enclosing branch.

## Exhaustiveness checking

Exhaustiveness is a required part of this proposal's type-tooling contract, inspired by [Flow's exhaustive checking](https://flow.org/en/docs/match/#exhaustive-checking). Every `v-match` analyzed by the template type checker must cover its subject type. A non-exhaustive match is an error by default, not an optional lint suggestion, and causes `vue-tsc --noEmit` to fail. Volar must show the corresponding diagnostic in the editor without requiring a separate ESLint rule or directive modifier.

### Coverage rules

The checker starts with the subject's type at the `v-match` site, after normal template ref unwrapping. It tracks the values that remain unhandled as it visits arms in source order:

1. An unguarded arm removes the values its pattern is proven to match. The match is exhaustive only when no values remain.
2. Literal and statically known singleton value patterns cover their respective values. A runtime value pattern whose type has multiple possible values cannot cover that entire type merely by comparing against its current value.
3. An or-pattern covers the union of its alternatives. Parentheses and `as` bindings do not change the underlying pattern's coverage.
4. Object and array patterns cover values recursively according to their runtime structure and length rules. Checking a tag only removes the entire tagged variant when its remaining subpatterns accept every value of that variant. Object rest does not narrow open-object coverage; array rest accepts all lengths at least as large as the listed prefix, provided that prefix matches. Binding a remainder does not change coverage compared with an unbound rest.
5. An unguarded top-level `_`, `const value`, or `_ as value` covers all remaining values. A wildcard nested in a structural pattern only covers its position, subject to the enclosing shape and property-presence checks.
6. An arm with an `if` guard removes no values from the remaining space. This also applies to `_ if (...)`, binding patterns with guards, and apparently complementary guards.

The required analyzable cases include finite literal unions, booleans, enum members with statically known values, discriminated object unions, finite combinations of nested object and tuple patterns, and array length partitions with trailing rest. Coverage must account for structural subcases, rather than only removing top-level union members.

`null` and `undefined` are distinct cases when present in the subject type. For example, `ref<Result>()` includes an initial `undefined` value and requires an `undefined` arm or an unconditional fallback in addition to all `Result` variants. An optional object property is not covered merely by matching its present value; the absent-property case must also be handled.

An unrestricted `string` or `number`, `any`, `unknown`, or a generic type whose constraints do not establish a covered value space requires an unguarded catch-all pattern. Finite literal arms cannot exhaust an open primitive type. If the checker cannot prove coverage for a more complex type or pattern combination, it must report that limitation and request a catch-all; it must not silently mark the match exhaustive. A subject typed as `never` has no remaining values and is vacuously covered.

### Missing-case diagnostics

The diagnostic is attached to the `v-match` subject expression. It identifies uncovered cases and suggests valid `v-when` patterns when they can be enumerated.

```vue
<script setup lang="ts">
defineProps<{ status: 'loading' | 'success' | 'error' }>()
</script>

<template>
  <!-- Error: non-exhaustive v-match; missing pattern 'error' -->
  <template v-match="status">
    <LoadingSpinner v-when="'loading'" />
    <SuccessPanel v-when="'success'" />
  </template>
</template>
```

Adding `<ErrorPanel v-when="'error'" />` completes coverage. Adding an unguarded `_` arm also completes coverage but intentionally handles future variants through the fallback. When authors want additions to a union to require a new branch, they should enumerate the variants without a catch-all.

A guarded variant still needs an unguarded arm:

```html
<template v-match="result">
  <ArticleView v-when="{ status: 'success', const data }" :article="data" />
  <RetryBanner
    v-when="{ status: 'error', const error } if (error.retriable)"
    :error="error"
  />
  <!-- Required to cover errors for which the guard is false -->
  <ErrorBanner v-when="{ status: 'error', const error }" :error="error" />
</template>
```

For a subject with success and error variants, removing the final arm must report the uncovered error variant. Neither complementary guards nor a final guarded wildcard discharge that requirement.

For nested patterns, the suggested missing case should be as specific as practical. If the subject is `['left' | 'right', 'top' | 'bottom']` and three combinations are covered, the diagnostic identifies the fourth tuple pattern. A pattern `{ status: 'success', data: null }` only covers successful results with `null` data; it cannot discharge all successful results if other data values are possible.

### Unreachable arms and intentional empty rendering

The checker also reports a warning on an arm whose entire pattern is proven unable to match any remaining value, either because earlier unguarded arms cover it or because it is outside the subject type. A guarded arm does not make a later arm unreachable merely by matching the same structural pattern. These warnings are separate from non-exhaustive-match errors; syntactically invalid wildcard placement remains a compiler error.

Intentionally rendering nothing is expressed by an empty arm, which still counts toward coverage:

```html
<template v-match="status">
  <SuccessPanel v-when="'success'" />
  <template v-when="_"></template>
</template>
```

The checker must analyze the arm before any optimization removes its empty render output. It must not infer that an omitted case means an intentional empty branch.

### Compiler boundary

The SFC compiler performs syntax and structural validation without requiring a TypeScript program. Semantic coverage diagnostics belong to Volar / vue-tsc, including JavaScript templates when the template checker has inferred types for them. Running only the template compiler or a build that does not run template type checking does not certify exhaustiveness.

If no runtime arm matches because type checking was skipped or the runtime value violates its declared type, the block renders nothing. This is Vue's conditional-rendering behavior, not Flow's match-expression exception behavior. The checker must not remove that runtime path or allow an exhaustiveness result to change client or SSR branch selection.

Compiler and tooling work may be developed in stages, but exhaustiveness checking with these default errors is part of the feature's acceptance criteria, not a deferred stretch goal.

# Drawbacks

- This adds new built-in directive syntax and a new pattern grammar.
- The compiler has to model a parent-child relationship between `v-match` and `v-when`.
- Pattern parsing is more complex than the previous strict-equality-only `v-case` draft.
- Adopting a shorthand would add a symbol to learn and require coordinated parser, formatter, and editor support. Each candidate has readability trade-offs described above.
- Required exhaustiveness errors and narrowing need coordinated Volar / vue-tsc support. Conservative analysis can require an explicit catch-all when coverage cannot be proven.
- Users may expect this to be identical to future JavaScript pattern matching. Vue should document that it is a template-level feature with an intentionally smaller initial surface.

# Alternatives

## Dedicated default syntax

`v-when.default` could identify a fallback separately. This RFC recommends `v-when="_"` instead: `_` is already a wildcard pattern, so it needs no separate modifier or fallback-only parsing rule. It also composes with guards and nested patterns, following Flow's wildcard spelling. Any shorthand should keep `_` in the attribute value, where every other pattern lives.

## Keep the previous `v-match` / `v-case` design

The earlier draft used:

```html
<template v-match="status">
  <p v-case="'loading'">Loading...</p>
  <p v-case="'error'">Error</p>
  <p v-case.default>Unknown</p>
</template>
```

This is simple, but it only covers strict equality and does not scale to object patterns, branch-local bindings, or guard expressions. It also misses the TC39 `when` vocabulary.

## Keep using `v-if` / `v-else-if`

The current directives remain fully supported and are still best when each branch checks unrelated conditions:

```html
<p v-if="isAdmin">Admin</p>
<p v-else-if="hasInvite">Invited</p>
<p v-else>Guest</p>
```

`v-match` is intended for branches that all inspect the same subject.

## Renderless `<Match>` component

```html
<Match :value="result">
  <When pattern="{ status: 'success', data: const article }">
    <ArticleView :article="article" />
  </When>
</Match>
```

This can be approximated in userland, but it cannot provide compiler-level binding scopes, branch-specific type narrowing, or the same optimization opportunities.

## Separate guard directive

Instead of `v-when="{ ... } if (...)"`, guards could be a second directive:

```html
<RetryBanner
  v-when="{ status: 'error', error: const error }"
  v-when-if="error.retriable"
/>
```

This is easier to parse but weaker as a pattern-matching story. Flow places guards on the arm, and TC39 models guards as part of the pattern. Keeping the guard inside `v-when` makes the branch read as one unit.

## Adopt all Flow patterns immediately

Flow includes instance patterns such as:

```js
match (shape) {
  Circle { const radius, ... } => radius,
  Square { const side, ... } => side,
}
```

Vue could eventually support class / component instance patterns, but they are not required for the initial template feature. Object, array, wildcard, binding, `|`, `as`, and guard patterns cover the common UI state cases with less runtime and parser complexity.

# Adoption strategy

- Existing `v-if` / `v-else-if` / `v-else` code continues to work. Any shorthand would reserve new attribute syntax; compatibility with existing literal attributes must be reviewed before adopting a symbol.
- Documentation should introduce `v-match` beside conditional rendering, emphasizing one-subject branching and branch-local bindings.
- Documentation should teach `v-when` first, then any adopted shorthand with the same semantics. Formatters and codemods should not force either spelling.
- ESLint rules can suggest `v-match` when a `v-if` chain repeatedly checks the same subject.
- A codemod can convert simple strict-equality chains to `v-match` / `v-when`.
- Implementation can proceed incrementally, but the completed feature must include the specified exhaustiveness errors, narrowing, and unreachable-arm warnings in Volar / vue-tsc.

# Unresolved questions

1. Should future rest patterns allow a nested pattern after `...`, beyond the initial unbound rest and `...const identifier` forms?
2. Should `let` bindings ever be supported in templates, or should Vue intentionally keep branch bindings immutable with `const` only?
3. Should Vue expose a public compiler AST node for pattern syntax so Volar, eslint-plugin-vue, and custom tooling can share the parser?
4. How should the pattern coverage analysis be shared between Volar / vue-tsc and optional lint rules while preserving the required error semantics?
5. Should runtime object pattern matching use `in` semantics, own-property semantics, or align exactly with the eventual ECMAScript proposal?
6. Should custom matcher protocols be considered later if TC39's `Symbol.customMatcher` advances?
7. Is there a shorthand that reads clearly as a match arm in attribute syntax? The current proposal keeps `v-when` long-form and defers symbol selection.
