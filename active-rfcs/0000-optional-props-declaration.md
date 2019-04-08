- Start Date: 2019-04-08
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Make component `props` declaration optional.

# Basic example

``` html
<template>
  <div>{{ $props.foo }}</div>
</template>

<script>
export default {}
</script>
```

# Motivation

In simple use cases where there is no need for runtime props type checking (especially in functional components), making props optional could result in simpler code.

# Detailed design

## Stateful Components

When a component has no `props` declarations, all attributes passed by the parent are exposed in `this.$props`. Unlike declared props, they will NOT be exposed directly on `this`. In addition, in this case `this.$attrs` and `this.$props` will be pointing to the same object.

``` html
<template>
  <div>{{ $props.foo }}</div>
</template>

<script>
export default {}
</script>
```

## Functional Components

This is based on plain-function functional components proposed in [Functional and Async Component API Change](TODO).

``` js
const FunctionalComp = props => {
  return h('div', props.foo)
}
```

To declare props for plain-function functional components, attach it to the function itself:

``` js
FunctionalComp.props = {
  foo: Number
}
```

Similar to stateful components, when props are declared, the `props` arguments will only contain the declared props - attributes received but not declared as props will be in the 3rd argument (`attrs`):

``` js
const FunctionalComp = (props, slots, attrs) => {
  // `attrs` contains all received attributes except declared `foo`
}

FunctionalComp.props = {
  foo: Number
}
```

For mode details on the new functional component signature, see [Render Function API Change](TODO).

# Drawbacks

N/A

# Alternatives

N/A

# Adoption strategy

The behavior is fully backwards compatible.
