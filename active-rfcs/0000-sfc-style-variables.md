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
          size: '2em'
        }
      }
    },
  }
</script>

<style>
  .text {
    color: var(--v-bind:color);

    /* shorthand + nested property access */
    font-size: var(--:font.size);
  }
</style>
```

# Motivation

Vue SFC styles provide straightforward CSS collocation and encapsulation, but it is purely static - which means up to this point we have no capability of dynamically updating the styles at runtime based on the component's state.

Now with [most modern browsers supporting native CSS variables](https://caniuse.com/#feat=css-variables), we can leverage it to easily connect the component's state and styles.

# Detailed design

The `<style>` tag in an SFC now supports CSS variables that start with the `v-bind:` prefix (or `:` as shorthand), for example:

```html
<style>
  .text {
    color: var(--v-bind:color);
    font-size: var(--:font.size);
  }
</style>
```

It should be noted that CSS variables names technically must be valid CSS identifiers which cannot contain characters like `:` or `.`. However, the syntax proposed in this RFC is used purely as a compile-time hint and is **not** included in the final runtime CSS. In terms of tooling integration, all major CSS parsers in the ecosystem can parse `var()` correctly with arbitrary inner content (verified on [ASTExplorer](https://astexplorer.net/)).

When such CSS variables are detected, the SFC compiler will perform the following:

1. Rewrite the variable to a hashed version. The above will be rewritten to:

   ```html
   <style>
     .text {
       color: var(--6b53742-color);
       font-size: var(--6b53742-font_size);
     }
   </style>
   ```

   Note the hashing will be applied in all cases, regardless of whether `<style>` tag is scoped or not. This means injected CSS variables will never accidentally leak into child components.

2. The corresponding variables will be injected to the component's root element as inline styles. For the example above, the final rendered DOM will look like this:

   ```html
   <div style="--6b53742-color:red;--6b53742-font_size:2em" class="text">
     hello
   </div>
   ```

   The injection is reactive - so if the component's `color` property changes, the injected CSS variable will be updated accordingly. This update is applied independent of the component's template update so changes to a CSS-only reactive property won't trigger template re-renders.

## Compilation Details

In order to inject the CSS variables, the compiler needs to generate and inject code like the following into the component's `setup()` function:

```js
import { useCssVars } from 'vue'

export default {
  setup() {
    // ...
    useCssVars(_ctx => ({
      color: _ctx.color,
      font_size: _ctx.font.size
    }))
  }
}
```

...where the `useCssVars` runtime helper sets up a `watchEffect` to reactively apply the variables to the DOM.

# Drawbacks

## Compilation cost

The compilation strategy requires the script compilation to do a parse of the `<style>` tag content first in order to determine the list of variables to expose. This will result in some duplicated CSS parsing cost.

In cases where no `v-bind:` variables are used, a quick regex check can skip a full parse, so the feature should have negligible impact on components that are not using it.

## Prettier integration

Currently, Prettier will attempt to format content inside `var(...)` when it contains colons or quotes, even thought technically it should not:

```css
.text {
  color: var(--v-bind:color);
  font-size: var(--:font.size);
}

/* Prettier will format the above to: */
.text {
  color: var(--v-bind: color);
  font-size: var(--: font.size);
}
```

Technically, the compilation strategy can handle this just fine by trimming the bound expression, but it can be a bit of an annoyance to users.

TODO: file prettier issue

# Adoption strategy

This is a fully backwards compatible new feature. However, we should make it clear that it relies on native CSS variables so the user needs to be aware of the browser support range.
