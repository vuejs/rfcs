- Start Date: 2020-06-29
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Support using component-state-driven CSS variables into Single File Components styles.

# Basic example

```html
<template>
  <div class="text">hello</div>
</template>

<script>
  export default {
    data() {
      return {
        color: 'red',
        font: {
          size: '2em',
        },
      }
    },
  }
</script>

<style>
  .text {
    color: v-bind(color);

    /* expressions (wrap in quotes) */
    font-size: v-bind('font.size');
  }
</style>
```

# Motivation

Vue SFC styles provide straightforward CSS collocation and encapsulation, but it is purely static - which means up to this point we have no capability of dynamically updating the styles at runtime based on the component's state.

Now with [most modern browsers supporting native CSS variables](https://caniuse.com/#feat=css-variables), we can leverage it to easily connect the component's state and styles.

# Detailed design

The `<style>` tag in an SFC now supports a custom CSS function named `v-bind`:

```html
<!-- in Vue SFC -->
<style>
  .text {
    color: v-bind(color);
  }
</style>
```

As expected, this would bind the `color` declaration's value to the `color` property of the component's state, reactively.

The `v-bind` function can support arbitrary JavaScript expressions inside, but since JavaScript expressions may contain characters that are not valid in CSS identifiers, they will need to be wrapped in quotes most of the time:

```css
.text {
  font-size: v-bind('theme.font.size');
}
```

When such CSS variables are detected, the SFC compiler will perform the following:

1. Rewrite the `v-bind()` to a native `var()` with a hashed variable name. The above will be rewritten to:

   ```css
    .text {
      color: var(--6b53742-color);
      font-size: var(--6b53742-theme_font_size);
    }
   ```

   Note the hashing will be applied in all cases, regardless of whether `<style>` tag is scoped or not. This means injected CSS variables will never accidentally leak into child components.

2. The corresponding variables will be injected to the component's root element as inline styles. For the example above, the final rendered DOM will look like this:

   ```html
   <div style="--6b53742-color:red;--6b53742-theme_font_size:2em;" class="text">
     hello
   </div>
   ```

   The injection is reactive - so if the component's `color` property changes, the injected CSS variable will be updated accordingly. This update is applied independent of the component's template update so changes to a CSS-only reactive property won't trigger template re-renders.

## Compilation Details

- In order to inject the CSS variables, the compiler needs to generate and inject code like the following into the component's `setup()` function:

  ```js
  import { useCssVars } from 'vue'

  export default {
    setup() {
      // ...
      useCssVars((_ctx) => ({
        color: _ctx.color,
        theme_font_size: _ctx.theme.font.size,
      }))
    },
  }
  ```

  ...where the `useCssVars` runtime helper sets up a `watchEffect` to reactively apply the variables to the DOM.

- The compilation strategy requires the script compilation to do a simple regex parse of the `<style>` tag content first in order to determine the list of variables to expose. However, this parse phase won't be as costly as a full AST-based parse.

- In production, the variable names can be further hashed to reduce CSS footprint:

   ```css
    .text {
      color: var(--x3b2fs2);
      font-size: var(--29fh29g);
    }
   ```

   Corresponding generated JavaScript code will be using the same hashes accordingly.

# Adoption strategy

This is a fully backwards compatible new feature. However, we should make it clear that it relies on native CSS variables so the user needs to be aware of the browser support range.
