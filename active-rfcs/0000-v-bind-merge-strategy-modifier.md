- Start Date: 2020-09-28
- Target Major Version: 3.x
- Reference Issues: [2059](https://github.com/vuejs/vue-next/issues/2059)
- Implementation PR: (leave this empty)

# Summary

Revert `v-bind` merge behaviour change to be the same as in Vue 2.
Add `.replace` modifier for `v-bind` to change default merge strategy.

# Basic example

```html
<div foo="bar" v-bind="{ foo: 'qux' }"></div>
<!--would render to-->
<div foo="bar"></div>
```

```html
<div foo="bar" v-bind.replace="{ foo: 'qux' }"></div>
<!--would render to-->
<div foo="qux"></div>
```

# Motivation

In Vue 2 `v-bind` would not replace already declared attributes or props on the element\component.
In Vue 3 `v-bind` that default merge strategy has been changed.
Now it _will_ replace anything that has been passed to `v-bind`.
Moreover, this behaviour depends on the `v-bind` placement order.
If you place `v-bind` directive **before** your attributes – it **will not** replace them and vice versa.

This change has some serious downsides:

* Reliance on attribute order is not an expected behaviour.
  We've learned that HTML and CSS never relies on attribute order and this change goes into conflict with that knowledge.
  
* Teams might have code standards that define specific order for
  attributes\directives\listeners for better readability and consistency.
  This change might be in conflict with these standards and introduce inconsistency into an already consistent codebase.

* Migration from Vue 2 will have a more complicated path.
  You'll have to check your whole codebase for `v-bind` placement order unless a sophisticated codemod is released.

In order to fix these issues this RFC focuses on making these 2 changes:

1. Revert `v-bind` merge behaviour to the one used in Vue 2.
2. Add a `.replace` modifier to `v-bind` to manually control merge behaviour.

## Why revert current behaviour?

Since Vue 3 is not yet widely adopted it is still possible to revert this change with minimal impact on current Vue 3 users.
At the time of writing this RFC the complete migration path from Vue 2 to Vue 3 is not completely ready,
and with this change in place we could skip another migration step. 

# Detailed design

`v-bind` directive placement order no longer affects its merge strategy. Instead, this strategy has to be changed explicitly.

By default `v-bind` will not replace attributes or props that have been already declared.
To change that `v-bind` should support a `.replace` modifier that will instruct compiler
to replace everything that's passed to a `v-bind` directive.

Given this template:

```html
<div foo="bar" v-bind="$attrs"></div>
<div foo="bar" v-bind.replace="$attrs"></div>
```

It should roughly compile to:

```js
[
  h('div', {
    ...this.$attrs,
    foo: 'bar',  
  }),
  h('div', {
    foo: 'bar',
    ...this.$attrs
  })
]
```

# Drawbacks

Users who already relay on that behaviour would have to go through a migration step.

# Alternatives


# Adoption strategy

It is to be discussed whether this change should be released as a minor update or not.

# Unresolved questions


