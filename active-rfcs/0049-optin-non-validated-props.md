- Start Date: 2026-03-07
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: (leave this empty)

# Summary

Introduce a runtime option `nonValidatedProps` that disables props filtering based on the `props` option declaration. When enabled (`true`), the runtime no longer distinguishes between `props` and `attrs` — all passed attributes are treated uniformly as props. This allows components (especially in JSX/Vapor) to accept props purely through TypeScript function signatures, without requiring a compile-time `props` option declaration.

Additionally, a type-level `propsVariance` configuration is introduced via module augmentation to control the variance of component props types, defaulting to `'contravariant'` (to preserve fallthrough attrs behavior) with an optional `'invariant'` mode for stricter type checking. This is purely a compile-time concern with no runtime footprint.

# Basic example

## Opting in to non-validated props

```ts
// app-level opt-in
const app = createApp(App)
app.config.nonValidatedProps = true
```

```ts
// per-component opt-in
export default defineComponent({
  nonValidatedProps: true,
  setup(props) {
    // all attributes passed to this component are available as props
    // no distinction between props and attrs
  },
})
```

## JSX Vapor with non-validated props

With `nonValidatedProps: true`, components in JSX/Vapor can define props purely as function type parameters:

```tsx
// Before: props option is required even in TSX
const MyComponent = defineComponent({
  props: {
    title: String,
    count: Number,
  },
  setup(props) {
    return () => <div>{props.title}: {props.count}</div>
  },
})

// After: with nonValidatedProps, props are just a type
const MyComponent = defineVaporComponent((props: { title: string; count: number }) => {
  return <div>{props.title}: {props.count}</div>
})

// Type-safe usage - compiler error if wrong props are passed
<MyComponent title="hello" count={42} />
<MyComponent title="hello" count="wrong" /> // Type error!
```

## Props variance control

Props variance is configured at the type level via module augmentation — no runtime setting is needed.

```ts
// Default behavior (no configuration needed): contravariant
// Extra attrs are allowed and fall through to the root element
```

```tsx
const MyComponent = defineVaporComponent((props: { title: string }) => {
  return <div>{props.title}</div>
})

// This is allowed - extra attrs fall through to the root element
<MyComponent title="hello" class="extra" id="my-id" />
```

To opt into strict (invariant) mode, add a type declaration:

```ts
// env.d.ts
declare module 'vue' {
  interface VuePropsConfig {
    variance: 'invariant'
  }
}
```

```tsx
const StrictComponent = defineVaporComponent((props: { title: string }) => {
  return <div>{props.title}</div>
})

// Type error - 'class' is not in the declared props
<StrictComponent title="hello" class="extra" /> // Type error!
```

# Motivation

## Problem 1: AST-based `defineProps` parsing is complex and fragile

The current Vue SFC compiler performs AST-level parsing of `defineProps` to extract prop declarations at compile time. This involves:

- Parsing TypeScript type literals and interfaces to derive runtime prop definitions.
- Resolving imported types, generic type parameters, and complex type expressions.
- Maintaining a separate type resolution system within the SFC compiler that mirrors (but cannot fully replicate) TypeScript's own type checker.

This approach is inherently limited. It cannot handle all valid TypeScript constructs (e.g., mapped types, conditional types, utility types referencing external modules). When it fails, developers encounter confusing compiler errors for code that is perfectly valid TypeScript.

## Problem 2: TSX requires the `props` option, hurting type ergonomics

In TSX (and JSX) usage, Vue components must declare a `props` option for the compiler to understand which attributes are props vs. attrs. This forces developers into a specific declaration style:

```tsx
// Forced to write this verbose declaration
defineComponent({
  props: {
    title: { type: String, required: true },
    count: { type: Number, default: 0 },
  },
  setup(props) { /* ... */ },
})
```

This is redundant when TypeScript already provides a complete type system for describing function parameters. The `props` option format — designed for runtime validation — is not a natural fit for TypeScript's structural type system.

## Problem 3: Poor compatibility with TypeScript's type system

Vue's runtime props system and TypeScript's type system have fundamental mismatches:

- **Runtime constructors vs. types:** `String`, `Number`, `Boolean` as runtime constructors map awkwardly to TypeScript types.
- **Validation vs. typing:** Runtime prop validation (e.g., `validator` function) has no type-level equivalent, leading to a dual-declaration problem.
- **Prop defaults and optional types:** The interaction between `default` and `required` in the props option creates complex type inference rules that are difficult to understand and maintain.

## Goal

By making props filtering opt-out, we can:

1. **Simplify the mental model:** Props are just function parameters. No special declaration format needed.
2. **Leverage TypeScript natively:** Props types come directly from TypeScript, with full IDE support, generic inference, and no custom type resolution.
3. **Enable simpler JSX/Vapor components:** Components become closer to plain functions, which is the natural model for Vapor's compiled output.
4. **Preserve backwards compatibility:** The default behavior (`nonValidatedProps: false`) is unchanged. Existing components work exactly as before.

## Attrs as contravariant props

In Vue, attributes not declared in `props` fall through as `attrs` to the component's root element. This is a useful feature — it allows parent components to pass HTML attributes (`class`, `style`, `id`, event listeners, etc.) without the child needing to explicitly declare each one.

When `nonValidatedProps` is enabled and there is no `props` declaration to filter against, **all** attributes are treated as props. The attrs fallthrough mechanism still works at the runtime level (undeclared props that match HTML attributes are applied to the root element).

At the type level, this means the props type should be **contravariant** by default: a component that declares `{ title: string }` should accept `{ title: string; class?: string; id?: string; ... }` without type errors. This preserves the attrs fallthrough ergonomics.

For use cases where strict prop typing is desired (e.g., library components that want to catch extraneous props at compile time), the `propsVariance: 'invariant'` option narrows the accepted props to exactly what is declared.

# Detailed design

## Runtime behavior

### `nonValidatedProps` option

A new boolean option, available at both the app level and per-component level:

```ts
interface AppConfig {
  // ... existing options
  nonValidatedProps: boolean // default: false
}

interface ComponentOptions {
  // ... existing options
  nonValidatedProps?: boolean
}
```

**When `nonValidatedProps` is `false` (default):**
- Current behavior is preserved.
- Props are filtered based on the `props` option declaration.
- Undeclared attributes go to `attrs`.

**When `nonValidatedProps` is `true`:**
- The runtime skips props validation and filtering.
- `instance.props` contains all attributes passed to the component (there is no filtering based on a props declaration).
- `instance.attrs` becomes an empty object (or is aliased to `instance.props`, implementation detail).
- The `props` option, if provided, is ignored for filtering purposes but can still be used for default values.

### Resolution order

Per-component `nonValidatedProps` takes precedence over the app-level setting:

```ts
app.config.nonValidatedProps = true

// This component still uses validated props
const LegacyComponent = defineComponent({
  nonValidatedProps: false, // overrides app-level setting
  props: { title: String },
  // ...
})
```

### Impact on `useAttrs()` and `$attrs`

When `nonValidatedProps` is `true`:

- `useAttrs()` returns an empty reactive object.
- `$attrs` in templates is an empty object.
- All attributes are accessible through `props` (or `$props` in templates).

This is a conscious trade-off. Components that rely on `$attrs` for attribute forwarding (e.g., `v-bind="$attrs"`) should either:
1. Keep `nonValidatedProps: false` (the default).
2. Use `v-bind="$props"` instead of `v-bind="$attrs"` to forward all received attributes.

## Type system design

### `propsVariance` — type-level configuration

`propsVariance` is a purely type-level concern with no runtime representation. It controls how the TypeScript compiler checks extra attributes passed to components. Configuration is done via module augmentation, following the same pattern as `vue-router`'s typed routes:

```ts
// Vue's type definition (in vue/types)
interface VuePropsConfig {
  // Users override this interface to change the default
  // variance: 'contravariant' | 'invariant'
}

type PropsVariance = VuePropsConfig extends { variance: infer V }
  ? V
  : 'contravariant' // default
```

**`'contravariant'` (default — no configuration needed):**

The component accepts any superset of the declared props type. Extra attributes are allowed and handled by the runtime (fallthrough to root element, or ignored).

Type-level behavior:

```ts
type ComponentProps<T> = T & Record<string, unknown>
```

This means:

```tsx
const Comp = defineVaporComponent((props: { title: string }) => { /* ... */ })

// All valid:
<Comp title="hello" />
<Comp title="hello" class="foo" />
<Comp title="hello" onClick={() => {}} />

// Invalid (required prop missing):
<Comp /> // Type error: 'title' is required

// Invalid (wrong type):
<Comp title={42} /> // Type error: 'title' must be string
```

**`'invariant'`:**

Opt in via module augmentation:

```ts
// env.d.ts
declare module 'vue' {
  interface VuePropsConfig {
    variance: 'invariant'
  }
}
```

The component accepts only the exact declared props type. Extra attributes cause a type error.

Type-level behavior:

```ts
type ComponentProps<T> = T // no excess property allowance
```

```tsx
const Comp = defineVaporComponent((props: { title: string }) => { /* ... */ })

// Valid:
<Comp title="hello" />

// Invalid:
<Comp title="hello" class="foo" /> // Type error: 'class' is not assignable
```

Note: `'invariant'` is a type-level-only constraint. At runtime, extra attributes are still received (since `nonValidatedProps` disables filtering) but are not used by the component. The type error serves as a development-time safeguard.

### Type inference for `defineVaporComponent`

```ts
function defineVaporComponent<P>(
  setup: (props: P) => VNode,
): Component<P>
```

The props type `P` is inferred directly from the `setup` function's parameter type. No additional type declaration is needed. The variance behavior is determined by the project-wide `VuePropsConfig` interface (resolved at the type level via module augmentation).

## Compiler changes

### SFC `<script setup>` with `nonValidatedProps`

When a component opts into `nonValidatedProps`, the SFC compiler behavior changes:

1. **`defineProps` becomes optional.** If `defineProps` is not used, all attributes are available as props.
2. **`defineProps` can use arbitrary TypeScript types.** Since the compiler no longer needs to extract runtime prop definitions from the type, complex types (mapped types, conditional types, imported types) work without restriction.
3. **No runtime props option is emitted.** The compiled component output does not include a `props` array or object.

```html
<script setup lang="ts">
// With nonValidatedProps: true (set at app level)
// defineProps is purely a type annotation — no AST parsing needed
const props = defineProps<{
  items: Map<string, { label: string; value: number }>
  transform: <T>(input: T) => T extends string ? number : string
}>()
</script>
```

### JSX / TSX

No compiler changes are needed for JSX/TSX. The type system handles everything:

- Component props types are inferred from the setup function or `defineProps` generic.
- JSX intrinsic element type checking uses the standard TypeScript JSX type resolution.

## Interaction with existing features

### `defineEmits`

`defineEmits` is unaffected. Event declarations remain separate from props.

### `withDefaults`

`withDefaults` can still be used with `defineProps` to provide default values, even with `nonValidatedProps: true`. The defaults are applied at the runtime level.

### `inheritAttrs`

When `nonValidatedProps` is `true`, `inheritAttrs` becomes a no-op (there are no attrs to inherit). Components that need attribute forwarding should use `v-bind="$props"`.

### Transition from `nonValidatedProps: false` to `true`

For components migrating to `nonValidatedProps: true`:

- Replace `v-bind="$attrs"` with `v-bind="$props"` (or explicitly bind relevant props).
- Replace `useAttrs()` with direct props access.
- The `props` option can be removed if it was only used for type checking (and not for runtime validation or defaults).

# Drawbacks

- **Two modes of operation.** Having both validated and non-validated props modes increases the surface area of Vue's component model. Documentation and teaching materials need to explain both.
- **`attrs` becomes less useful.** With `nonValidatedProps: true`, the attrs concept effectively disappears. Components and libraries that rely on attrs separation (e.g., UI component libraries that use `$attrs` for HTML attribute forwarding) cannot use this mode without refactoring.
- **Potential ecosystem fragmentation.** If some libraries use `nonValidatedProps: true` and others don't, interoperability could be affected. Clear conventions are needed.
- **Runtime validation loss.** The `props` option provides runtime validation (type checking, required checks, custom validators) that is valuable during development. With `nonValidatedProps: true`, these checks are skipped, and developers must rely solely on TypeScript for type safety.
- **Variance concept may be unfamiliar.** The `propsVariance` option introduces type theory terminology that may confuse developers unfamiliar with variance concepts.

# Alternatives

## 1. Improve `defineProps` type resolution

Instead of bypassing props validation, invest in making the SFC compiler's TypeScript type resolution more complete. This addresses Problem 1 but not Problems 2 or 3, and has diminishing returns as TypeScript's type system continues to grow in complexity.

## 2. Compiler-only approach (no runtime change)

Make the compiler smarter about TSX/JSX props without changing the runtime behavior. The compiler could emit a full `props` option from TypeScript types, handling the translation automatically. This maintains runtime validation but still requires the compiler to understand arbitrary TypeScript types.

## 3. `defineProps` with pass-through mode

Instead of a separate option, add a mode to `defineProps` that disables filtering:

```ts
const props = defineProps<MyProps>({ passthrough: true })
```

This is more localized but doesn't address the JSX/Vapor use case where `defineProps` itself is the friction point.

## 4. Separate component type for "function components"

Introduce a new component definition API specifically for simple function components:

```ts
const MyComponent = defineFunctionComponent((props: { title: string }) => {
  return <div>{props.title}</div>
})
```

This avoids changing existing behavior but fragments the component model further.

# Adoption strategy

- **Non-breaking change.** The default behavior (`nonValidatedProps: false`) is preserved. Existing applications are unaffected.
- **Incremental opt-in.** Developers can enable `nonValidatedProps` at the app level for new projects, or per-component for gradual migration.
- **Recommended for Vapor.** The `nonValidatedProps: true` mode is the recommended default for Vapor-based applications, where components are closer to plain functions.
- **Migration guide.** Provide documentation on:
  - How to migrate from `props` option to TypeScript-only props.
  - How to replace `$attrs` usage with `$props`.
  - When to configure `VuePropsConfig` with `'invariant'` vs. the default `'contravariant'`.
- **Tooling updates.** Volar and eslint-plugin-vue need to support the `nonValidatedProps` mode:
  - Skip props validation diagnostics when `nonValidatedProps` is `true`.
  - Support `VuePropsConfig` module augmentation for JSX type checking.
- **Codemod.** A codemod can be provided to:
  - Add `nonValidatedProps: true` to component options.
  - Replace `$attrs` with `$props`.
  - Convert `props` option declarations to TypeScript type parameters.

# Unresolved questions

1. **Naming:** Is `nonValidatedProps` the best name? Alternatives include `rawProps`, `untypedProps`, `skipPropsValidation`, or `propsMode: 'passthrough'`. The name should clearly convey that props filtering is disabled.
2. **Default for Vapor:** Should Vapor components default to `nonValidatedProps: true`? This would make Vapor's default behavior different from the VDOM runtime, but it aligns with Vapor's functional component model.
3. **Attrs forwarding pattern:** With `nonValidatedProps: true`, what is the recommended pattern for attribute forwarding in wrapper components? Should a new utility (e.g., `splitProps()`) be provided to separate "own" props from "forwarded" props?
4. **Runtime validation opt-in:** For developers who want both `nonValidatedProps: true` (no filtering) and runtime validation, should there be a way to add validation back (e.g., via a `validate` option or a `withValidation()` wrapper)?
5. **Interaction with `defineModel`:** How does `defineModel` behave when `nonValidatedProps` is `true`? The `modelValue` prop is implicitly declared — should it still be special-cased?
6. **Variance naming:** Is `VuePropsConfig` with `'contravariant'` / `'invariant'` too academic? Alternatives like `'open' | 'strict'` or a simple `strictProps: boolean` interface might be more approachable.
7. **Partial opt-in:** Should there be a way to mark only specific props as "validated" while leaving the rest unfiltered? This would allow a middle ground between full validation and no validation.
