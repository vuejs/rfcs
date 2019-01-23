- Start Date: 01-16-2019
- Target Major Version: (2.x / 3.x)
- Reference Issues: https://github.com/vuejs/rfcs/pull/2
- Implementation PR: (leave this empty)

# Summary

Add shorthand syntax for the `v-slot` syntax proposed in [rfc-0001](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0001-new-slot-syntax.md). Please make sure to read it first to get proper context for this proposal.

# Basic example

``` html
<foo>
  <template #header="{ msg }">
    Message from header: {{ msg }}
  </template>

   <template #footer>
    A static footer
  </template>
</foo>
```

# Motivation

A shorthand, as the name entails, primarily aims to provide more succinct syntax.

In Vue we currently provide shorthands only for two directives: `v-bind` and `v-on`:

``` html
<div v-bind:id="id"></div>
<div :id="id"></div>

<button v-on:click="onClick"></button>
<button @click="onClick"></button>
```

`v-bind` and `v-on` often get used repeatedly on the same element, and can get verbose when the information differing between each is **the directive argument** instead of the directive itself. **Shorthands thus help improve the signal/noise ratio by shortening the parts that are duplicated (`v-xxx`) and highlighting the parts that are different (the arguments).**

The new `v-slot` syntax can also get the same problem in components that expect multiple slots:

``` html
<TestComponent>
  <template v-slot:one="{ name }">Hello {{ name }}</template>
  <template v-slot:two="{ name }">Hello {{ name }}</template>
  <template v-slot:three="{ name }">Hello {{name }}</template>
</TestComponent>
```

Here `v-slot` is repeated multiple times where the actual differing part is the slot name (denoted by the argument).

The shorthand helps slot names to be more easily scan-able:

``` html
<TestComponent>
  <template #one="{ name }">Hello {{ name }}</template>
  <template #two="{ name }">Hello {{ name }}</template>
  <template #three="{ name }">Hello {{name }}</template>
</TestComponent>
```

# Detailed design

The shorthand follows very similar rules to the shorthand of `v-bind` and `v-on`: replace the directive name plus the colon with the shorthand symbol (`#`).

``` html
<!-- full syntax -->
<foo>
  <template v-slot:header="{ msg }">
    Message from header: {{ msg }}
  </template>

   <template v-slot:footer>
    A static footer
  </template>
</foo>

<!-- shorthand -->
<foo>
  <template #header="{ msg }">
    Message from header: {{ msg }}
  </template>

   <template #footer>
    A static footer
  </template>
</foo>
```

**Similar to `v-bind` and `v-on`, the shorthand only works when an argument is present. This means `v-slot` without an argument cannot be simplified to `#=`.** For default slots, either the full syntax (`v-slot`) or an explicit name (`#default`) should be used.

``` html
<foo v-slot="{ msg }">
  {{ msg }}
</foo>

<foo #default="{ msg }">
  {{ msg }}
</foo>
```

The choice of `#` is a selection based on the feedback collected in the previous RFC. It holds a resemblance to the id selector in CSS and translates quite well conceptually to stand for a slot's name.

Some example usage in a real-world library that relies on scoped slots ([vue-promised](https://github.com/posva/vue-promised)):

 ``` html
<Promised :promise="usersPromise">
  <template #pending>
    <p>Loading...</p>
  </template>

   <template #default="users">
    <ul>
      <li v-for="user in users">{{ user.name }}</li>
    </ul>
  </template>

   <template #rejected="error">
    <p>Error: {{ error.message }}</p>
  </template>
</Promised>
```

# Drawbacks

- Some may argue that slots are not that commonly used and therefore don't really need a shorthand, and it could result in extra learning curve for beginners. In response to that:

  1. I believe scoped slot is an important mechanism for building 3rd party component suites that are highly customizable and composable. I think we will see more component libraries relying on slots for customization and composition in the future. For a user using such a component library, a shorthand will become quite valuable (as demonstrated in the examples).

  2. The translation rules for the shorthand is straightforward and consistent with existing shorthands. If the user learns how the base syntax works, understanding the shorthand is a minimal extra step.

# Alternatives

Some alternative symbols have been presented and discussed in the previous RFC. The only other one with similarly positive interest is `&`:

``` html
<foo>
  <template &header="{ msg }">
    Message from header: {{ msg }}
  </template>

   <template &footer>
    A static footer
  </template>
</foo>
```

# Adoption strategy

This should be a natural extension on top of the new `v-slot` syntax. Ideally, we'd want introduce the base syntax and the shorthand at the same time so users learn both at the same time. Introducing the shorthand at a later time runs the risk of some users only aware of the `v-slot` syntax and get confused when seeing the shorthand in others' code.

# Unresolved questions

N/A
