- Start Date: 2019-11-29
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- Rename the `v-enter` transition class to `v-enter-from`
- Rename the `v-leave` transition class to `v-leave-from`

# Basic example

``` css
/* before */
.v-enter, .v-leave-to{
  opacity: 0;
}

/* after */
.v-enter-from, .v-leave-to{
  opacity: 0;
}
```

# Motivation

Before v2.1.8, we only had two transition classes each transition direction. For example for the enter transition, we had `v-enter` and `v-enter-active`. In v2.1.8, we introduced `v-enter-to` to address the [timing gap between enter/leave transitions](https://github.com/vuejs/vue/issues/4510), however, for backwards compatibility the `v-enter` name was untouched:

``` css
.v-enter, .v-leave-to {
  opacity: 0;
}
.v-leave, .v-enter-to {
  opacity: 1
}
```

The asymmetry and lack of explicitness in `.v-enter` and `.v-leave` makes these classes a bit mind bending to read and understand. This is why we are proposing to change the above to:

``` css
.v-enter-from, .v-leave-to {
  opacity: 0;
}
.v-leave-from, .v-enter-to {
  opacity: 1
}
```

...which better indicates what state these classes apply to.

# Detailed design

- `.v-enter` is renamed to `.v-enter-from`
- `.v-leave` is renamed to `.v-leave-from`
- The `<transition>` component's related prop names are also changed:
  - `leave-class` is renamed to `leave-from-class` (in render functions or JSX, can be written as `leaveFromClass`)
  - `enter-class` is renamed to `enter-from-class` (in render functions or JSX, can be written as `enterFromClass`)
  - `appear-class` is renamed to `appear-from-class` (in render functions or JSX, can be written as `appearFromClass`)

# Adoption strategy

Old class names can be easily supported in the compat build, with warnings to guide migration.
