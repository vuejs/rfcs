- Start Date: 2020-03-25
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/rfcs/pull/28

# Summary

Introduce a dedicated API for defining async components.

# Basic example

```js
import { defineAsyncComponent } from "vue"

// simple usage
const AsyncFoo = defineAsyncComponent(() => import("./Foo.vue"))

// with options
const AsyncFooWithOptions = defineAsyncComponent({
  loader: () => import("./Foo.vue"),
  loading: LoadingComponent,
  error: ErrorComponent,
  delay: 200,
  timeout: 3000
})
```

# Motivation

Per changes introduced in [RFC-0008: Render Function API Change](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0008-render-function-api-change.md), in Vue 3 plain functions are now treated as functional components. Async components must now be explicitly defined via an API method.

# Detailed design

## Simple Usage

```js
import { defineAsyncComponent } from "vue"

// simple usage
const AsyncFoo = defineAsyncComponent(() => import("./Foo.vue"))
```

`defineAsyncComponent` can accept a loader function that returns a Promise resolving to the actual component.

- If the resolved value is an ES module, the `default` export of the module will automatically be used as the component.

- **Difference from 2.x:** Note that the loader function no longer receives the `resolve` and `reject` arguments like in 2.x - a Promise must always be returned.

  For code that relies on custom `resolve` and `reject` in the loader function, the conversion is straightforward:

  ```js
  // before
  const Foo = (resolve, reject) => {
    /* ... */
  }

  // after
  const Foo = defineAsyncComponent(() => new Promise((resolve, reject) => {
    /* ... */
  }))
  ```

## Options Usage

```js
import { defineAsyncComponent } from "vue"

const AsyncFooWithOptions = defineAsyncComponent({
  loader: () => import("./Foo.vue"),
  loading: LoadingComponent,
  error: ErrorComponent,
  delay: 100, // default: 200
  timeout: 3000, // default: Infinity
  suspensible: false // default: true
})
```

Options except `loader` and `suspensible` works exactly the same as in 2.x.

**Difference from 2.x:**

The `component` option is replaced by the new `loader` option, which accepts the same loader function as in the simple usage.

In 2.x, an async component with options is defined as

```ts
() => ({
  component: Promise<Component>
  // ...other options
})
```

Whereas in 3.x it is now:

```ts
defineAsyncComponent({
  loader: () => Promise<Component>
  // ...other options
})
```

## Using with Suspense

Async component in 3.x are *suspensible* by default. This means if it has a `<Suspense>` in the parent chain, it will be treated as an async dependency of that `<Suspense>`. In this case, the loading state will be controlled by the `<Suspense>`, and the component's own `loading`, `error`, `delay` and `timeout` options will be ignored.

The async component can opt-out of Suspense control and let the component always control its own loading state by specifying `suspensible: false` in its options.

# Adoption strategy

- The syntax conversion is mechanical and can be performed via a codemod. The challenge is in determining which plain functions should be considered async components. Some basic heuristics can be used:

  - Arrow functions that returns dynamic `import` call to `.vue` files
  - Arrow functions that returns an object with the `component` property being a dynamic `import` call.

  Note this may not cover 100% of the existing usage.

- In the compat build, it is possible to check the return value of functional components and warn legacy async components usage. This should cover all Promise-based use cases.

- The only case that cannot be easily detected is 2.x async components using manual `resolve/reject` instead of returning promises. Manual upgrade will be required for such cases but they should be relatively rare.
