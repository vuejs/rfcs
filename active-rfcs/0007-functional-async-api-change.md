- Start Date: 2019-04-08
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- Functional components must be written as plain functions
  - `{ functional: true }` option removed
  - `<template functional>` no longer supported
- Async component must now be created via a dedicated API method

# Basic example

``` js
import { h } from 'vue'

const FunctionalComp = props => {
  return h('div', `Hello! ${props.name}`)
}
```

``` js
import { defineAsyncComponent } from 'vue'

const AsyncComp = defineAsyncComponent(() => import('./Foo.vue'))
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

const FunctionalComp = (props, { slots, attrs, emit }) => {
  return h('div', `Hello! ${props.name}`)
}
```

- The `functional` option is removed, and object format with `{ functional: true }` is no longer supported.

- SFCs will no longer support `<template functional>` - if you need anything more than a function, just use a normal component.

- The function signature has also changed:
  - `h` is now imported globally.
  - The function receives two arguments: `props` and a context object that exposes `slots`, `attrs` and `emit`. These are equivalent to their `$`-prefixed equivalents on a stateful component.

## Comparison with Old Syntax

The new function arguments should provide the ability to fully replace the [2.x functional render context](https://vuejs.org/v2/guide/render-function.html#Functional-Components):

- `props` and `slots` have equivalent values;
- `data` and `children` are no longer necessary (just use `props` and `slots`);
- `listeners` will be included in `attrs`;
- `injections` can be replaced using the new `inject` API (part of [Composition API](https://vue-composition-api-rfc.netlify.com/api.html#provide-inject)):

  ``` js
  import { inject } from 'vue'
  import { themeSymbol } from './ThemeProvider'

  const FunctionalComp = props => {
    const theme = inject(themeSymbol)
    return h('div', `Using theme ${theme}`)
  }
  ```

- `parent` access will be removed. This was an escape hatch for some internal use cases - in userland code, props and injections should be preferred.

## Optional Props Declaration

To make it easier to use for simple cases, 3.x functional components do not require `props` to be declared:

```js
const Foo = props => h('div', props.msg)
```
``` html
<Foo msg="hello!" />
```

With no explicit props declaration, the first argument `props` will contain everything passed to the component by the parent.

## Explicit Props Declaration

To add explicit props declarations, attach `props` to the function itself:

``` js
const FunctionalComp = props => {
  return h('div', `Hello! ${props.name}`)
}

FunctionalComp.props = {
  name: String
}
```

## Async Component Creation

The new async component API is discussed in [its own dedicated RFC](https://github.com/vuejs/rfcs/pull/148).

# Drawbacks

- Migration cost

# Alternatives

N/A

# Adoption strategy

- For functional components, a compatibility mode can be provided for one-at-a-time migration.

- SFCs using `<template functional>` should be converted to normal SFCs.
