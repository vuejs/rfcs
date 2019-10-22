- Start Date: 2019-10-22
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/vue/issues/10583
- Implementation PR: (leave this empty)

# Summary

Rename `:key` (i.e. `v-bind:key`) to `v-key`

# Basic example

<li v-for="product in products" v-key="product.id">

# Motivation

The current syntax of `key` ( https://vuejs.org/v2/guide/list.html#Maintaining-State ) is inconsistent with the `v-bind` directive:
https://vuejs.org/v2/guide/syntax.html#Attributes says that `<div v-bind:foo="...">` will result in `<div foo="...">`.

However, `v-bind:key` (shorthand: `:key`) will not bind anything to the HTML attribute `key`! The result will _not_ be `<div key="...">`.

So I suggest to just rename `key` to `v-key`, to show that this is something _internal_ to Vue.

Besides its inconsistency, the current implementation breaks the ability to actually introduce an attribute named `key`.

# Detailed design

# Drawbacks

# Alternatives

Leave it as it is :-)

# Adoption strategy

For a transition period, support both syntaxes, and log a deprecation warning to the console.

# Unresolved questions
