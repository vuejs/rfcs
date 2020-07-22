- Start Date: 2020-07-22
- Target Major Version: 2.x and 3.x
- Reference Issues:
- Implementation PR:

# Summary

Allow to define named `slots` and `argument` in the `typescript` to provide auto-completion and type inference.

# Basic example

```ts
// named slots
defineComponent({
  slots: {
    // slot name `item`
    item: { value: Number },
  },
  // ...
})
```

# Motivation

Having type validation when using `slots`.

This will allow to have type inference with using render funcions `h` and it will allow extensions (`vetur`, etc) to be able to provide the correct types on `SFC`

# Detailed design

Implementation will be similar to `emit` typings. This can also be used at the run-time to validate the slot as we do with props.

# Drawbacks

This is optional, for large applications this will be useful.

# Alternatives

There's `web-types.json` (Jetbrains) that describes the slots. AFAIK no solution for `Vetur`

# Adoption strategy

# Unresolved questions

Should we do a pure typescript or allow to do similar prop validation?
