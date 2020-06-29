- Start Date: 2020-06-29
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Support injecting component-state-driven CSS variables into Single File Components styles.

# Basic example

```html
<template>
  <div class="text">hello</div>
</template>

<script>
export default {
  data() {
    return {
      color: 'red'
    }
  }
}
</script>

<style :vars="{ color }">
.text {
  color: var(--color);
}
</style>
```

# Motivation

Vue SFC styles provide straightforward CSS collocation and encapsulation, but it is purely static - which means up to this point we have no capability of dynamically updating the styles at runtime based on the component's state.

Now with [most modern browsers supporting native CSS variables](https://caniuse.com/#feat=css-variables), we can leverage it to easily connect the component's state and styles.

# Detailed design

The `<style>` tag in an SFC now supports a `:vars` binding, which accepts an expression for the key/values to inject as CSS variables. The expression must be an object literal and cannot contain computed keys (the compiler will warn such cases - see reasoning in the next section). Other than that, it is evaluated in the same context as expressions inside `<template>`.

The variables will be applied to the component's root element as inline styles. In the above example, given a `:vars` binding that evaluates to `{ color: 'red' }`, the rendered HTML will be:

```html
<div style="--color:red" class="text">hello</div>
```

## Usage with `<style scoped>`

When `:vars` is used with `<style scoped>`, we need to make sure the CSS variables do not leak to descendent components or accidentally shadow CSS variables higher up the DOM tree. The applied CSS variables will be prefixed with the component's scope ID:

```html
<div style="--v-6b53742-color:red" class="text">hello</div>
```

Similarly, CSS variables inside `<style>` that matches the keys in the `:vars` expression will also be rewritten accordingly. This is also why we require the `:vars` expression to be an object literal with no computed keys.

# Alternatives

## Auto Detect Variables

One of the considered alternatives is auto detecting variables to inject using a naming convention:

```html
<style>
.text {
  color: var(--v-color);
}
</style>
```

However, this has a few disadvantages:

- Intention is less explicit;

- Requires a full PostCSS parse first to extract matching variables before the main script can be processed. With `:vars` being declared on the `<style>` tag, the compiler can process the `<script>` part without parsing the CSS at all.

# Adoption strategy

This is a fully backwards compatible new feature. However, we should make it clear that it relies on native CSS variables so the user needs to be aware of the browser support range.
