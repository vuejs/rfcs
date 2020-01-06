- Start Date: 2019-11-29
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Using `<transition>` as component's root will no longer trigger transitions when the component is toggled from the outside.

# Basic example

Before:

``` html
<!-- modal component -->
<template>
  <transition>
    <div class="modal"><slot/></div>
  </transition>
</template>

<!-- usage -->
<modal v-if="showModal">hello</modal>
```

After: expose a prop to control the toggle

``` html
<!-- modal component -->
<template>
  <transition>
    <div v-if="show" class="modal"><slot/></div>
  </transition>
</template>

<!-- usage -->
<modal :show="showModal">hello</modal>
```

# Motivation

The current 2.x behavior worked by accident, but with some quirks. Instead of disabling the usage, we added more fix to make it work because some users were relying on the behavior. However, semantically this usage does not make sense: by definition, the `<transition>` component works by reacting to the toggling of its inner content, not the toggling of itself:

``` html
<!-- this does not work -->
<transition v-if="show">
  <div></div>
</transition>

<!-- this is expected usage -->
<transition>
  <div v-if="show"></div>
</transition>
```

In supporting the 2.x behavior, it also created a number of complexities when it comes to determining the `appear` status of the transition.

# Detailed design

In 3.0, toggling a component with `<transition>` as its root node will no longer trigger the transition. Instead, the component should expose a boolean prop to control the presence of the content inside `<transition>`.

# Drawbacks

The old behavior cannot be supported simultaneously with the new one in the compat build.

# Adoption strategy

Reliance on the old behavior can be found through static analysis by detecting root `<transition>` components with no `v-if` or `v-show` directives on its inner content. The migration tool can then guide the user to upgrade these cases.
