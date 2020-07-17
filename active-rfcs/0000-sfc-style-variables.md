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

<style vars="{ color }">
.text {
  color: var(--color);
}
</style>
```

# Motivation

Vue SFC styles provide straightforward CSS collocation and encapsulation, but it is purely static - which means up to this point we have no capability of dynamically updating the styles at runtime based on the component's state.

Now with [most modern browsers supporting native CSS variables](https://caniuse.com/#feat=css-variables), we can leverage it to easily connect the component's state and styles.

# Detailed design

The `<style>` tag in an SFC now supports a `vars` binding, which accepts an expression for the key/values to inject as CSS variables. It is evaluated in the same context as expressions inside `<template>`.

The variables will be applied to the component's root element as inline styles. In the above example, given a `vars` binding that evaluates to `{ color: 'red' }`, the rendered HTML will be:

```html
<div style="--color:red" class="text">hello</div>
```

## Usage with `<style scoped>`

When `vars` is used with `<style scoped>`, we need to make sure the CSS variables do not leak to descendent components or accidentally shadow CSS variables higher up the DOM tree. The applied CSS variables will be prefixed with the component's scope ID:

```html
<div style="--6b53742-color:red" class="text">hello</div>
```

Similarly, CSS variables inside `<style>` will also need be rewritten accordingly. With the following code:

```html
<style scoped vars="{ color }">
h1 {
  color: var(--color);
}
</style>
```

The inner CSS will be compiled into:

```css
h1 {
  color: var(--6b53742-color);
}
```

**Note that when `scoped` and `vars` are both present, all CSS variables are considered to be local.** In order to reference a global CSS variable here, use the `global:` prefix:

```html
<style scoped vars="{ color }">
h1 {
  color: var(--color);
  font-size: var(--global:fontSize);
}
</style>
```

The above compiles into:

```css
h1 {
  color: var(--6b53742-color);
  font-size: var(--fontSize);
}
```

When there is only `scoped` and no `vars`, CSS variables are untouched. This preserves backwards compatibility.

# Adoption strategy

This is a fully backwards compatible new feature. However, we should make it clear that it relies on native CSS variables so the user needs to be aware of the browser support range.
