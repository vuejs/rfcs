- Start Date: 2019-04-08
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- Functional components must be written as plain functions
  - `{ functional: true }` option removed
  - `<template functional>` no longer supported
- Async component must be created via the `createAsyncComponent` API method

# Basic example

``` js
import { h } from 'vue'

const FunctionalComp = props => {
  return h('div', `Hello! ${props.name}`)
}
```

``` js
import { createAsyncComponent } from 'vue'

const AsyncComp = createAsyncComponent(() => import('./Foo.vue'))
```

# Motivation

## Simplify Functional Components

In 2.x, functional components must be created using the following format:

``` js
const FunctionalComp = {
  functional: true,
  render(h) {
    return h('div', `Hello! ${props.name}`)
  }
}
```

This has the following issues:

- Even when the component needs nothing but the render function, it still needs to use the object with `functional: true`.

- Some options are supported (e.g. `props` and `inject`) but others are not (e.g. `components`). However, users often expect all options to be supported because it looks so similar to a normal stateful component (especially when they use SFC with `<template functional>`).

Another aspect of the problem is that we've noticed some users are using functional components solely for performance reasons, e.g. in SFCs with `<template functional>`, and are requesting us to support more stateful component options in functional components. However, I don't think this is something we should invest more time in.

In v3, the performance difference between stateful and functional components has been drastically reduced and will be insignificant in most use cases. As a result there is no longer a strong incentive to use functional components just for performance, which also no longer justifies the maintenance cost of supporting `<template functional>`. Functional components in v3 should be used primarily for simplicity, not performance.

# Detailed Design

In 3.x, we intend to support functional components **only** as plain functions:

``` js
import { h } from 'vue'

const FunctionalComp = (props, slots) => {
  return h('div', `Hello! ${props.name}`)
}
```

- The `functional` option is removed, and object format with `{ functional: true }` is no longer supported.

- SFCs will no longer support `<template functional>` - if you need anything more than a function, just use a normal component.

- The function signature has also changed - `h` is now imported globally. Instead of a render context, props and slots and other values are passed in. For more details on how the new arguments can replace 2.x functional render context, see the [Render Function API Change RFC](https://github.com/vuejs/rfcs/pull/28).

## Runtime Props Validation

Props declaration is now optional (only necessary when runtime validation is needed). To add runtime validation or default values, attach `props` to the function itself:

``` js
const FunctionalComp = props => {
  return h('div', `Hello! ${props.name}`)
}

FunctionalComp.props = {
  name: String
}
```

## Async Component Creation

With the functional component change, Vue's runtime won't be able to tell whether a function is being provided as a functional component or an async component factory. So in v3 async components must now be created via a new API method:

``` js
import { createAsyncComponent } from 'vue'

const AsyncComp = createAsyncComponent(() => import('./Foo.vue'))
```

The method also supports advanced options:

``` js
const AsyncComp = createAsyncComponent({
  factory: () => import('./Foo.vue'),
  delay: 200,
  timeout: 3000,
  error: ErrorComponent,
  loading: LoadingComponent
})
```

This will make async component creation a little more verbose, but async component creation is typically a low-frequency use case, and are often grouped in the same file (the routing configuration).

# Drawbacks

- Migration cost

# Alternatives

N/A

# Adoption strategy

- For functional components, a compatibility mode can be provided for one-at-a-time migration.

- For async components, the migration is straightforward and we can emit warnings when function components return Promise instead of VNodes.

- SFCs using `<template functional>` should be converted to normal SFCs.
