- Start Date: 2019-11-08
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- Drop support for using numbers (keyCodes) as `v-on` modifiers
- Remove `config.keyCodes`

# Basic example

N/A

# Motivation

In Vue 2.x, `v-on` already supports using the kebab-case version of any valid `KeyboardEvent.key` as a modifier. For example, to trigger the handler only when `event.key === 'PageDown'`:

``` html
<input @keyup.page-down="onArrowUp">
```

This makes number keyCodes and `config.keyCodes` redundant. In addition, [`KeyboardEvent.keyCode` has been deprecated](https://developer.mozilla.org/en-US/docs/Web/API/KeyboardEvent/keyCode), so it would make sense for Vue to stop supporting it as well.

# Drawbacks

N/A

# Alternatives

N/A

# Adoption strategy

- A codemod can detect usage of number `keyCode` modifier usage and convert it to `key` equivalents.

- In compat build, `config.keyCode` can be supported, and the runtime can emit warning when a keyCode alias is matched to allow easy migration.
