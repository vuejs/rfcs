- Start Date: 2022-05-17
- Target Major Version: 3.x
- Reference Issues:
- Implementation PR:

# Summary

Add `fragment` built-in component that acts as a `<template>` empty wrapper tag.

# Basic example

## Conditional wrapper component

```html
<template>
  <WrapperComponent v-if="shouldWrap">
    <img src="cat.jpg" alt="Cat" />
  </WrapperComponent>
  <img v-else src="cat.jpg" alt="Cat" />
</template>
```

## `Fragment` component

```html
<template>
  <component :is="shouldWrap ? WrapperComponent : 'fragment'">
    <img src="cat.jpg" alt="Cat" />
  </component>
</template>
```

# Motivation

There are cases when some markup should be conditionally wrapped within a tag\component or not wrapped at all. Right now you have 2 options on how to deal with that: either duplicate the markup or extract that code into a separate component. Both are not ideal: duplicate code is invisibly coupled (changes in one copy should be reflected in all other copies), extracting into component is cumbersome. It gets more tedious when you have multiple of those cases in a single component.

# Detailed design

`<component is="fragment">` should be compiled into `h('fragment')`. Vue renderer should be updated accordingly to support rendering fragments that way.

# Drawbacks

Possibly a duplicate of `<template>` tag functionality. See **Unresolved questions** section.

# Alternatives

Another approach could be to add suppoort for `null` or `undefined` as the `<component>` `is` attribute.

# Adoption strategy

This is not a breaking change. All the exisiting components that already define a `Fragment` component should continue to function without any issues.

# Unresolved questions

Should we allow for `<fragment>` tag to work the same way?

Should it be `<>` instead of `<fragment>`?

Would that conflict with the `<template>` tag?
