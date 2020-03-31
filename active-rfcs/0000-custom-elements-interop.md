- Start Date: 2020-03-25
- Target Major Version: 3.x
- Reference Issues: N/A

# Summary

- **Breaking:** Custom Elements whitelisting is now performed during template compilation, and should be configured via compiler options instead of runtime config.

- **Breaking:** Restrict the special `is` prop usage to the reserved `<component>` tag only.

- Introduce a new `v-is` directive to support 2.x use cases where `is` was used on native elements to work around native HTML parsing restrictions.

# Motivation

- Provide native custom element support in a more performant fashion
- Improve support for [customized built-in elements](https://html.spec.whatwg.org/multipage/custom-elements.html#custom-elements-customized-builtin-example).

# Detailed design

## Autonomous Custom Elements

In Vue 2.x, whitelisting tags as custom elements is done via `Vue.config.ignoredElements`. The downside is that when this config option is used, a check needs to be performed on every vnode creation call.

**In Vue 3.0, this check is performed during template compilation.** For example, given the following template:

```html
<plastic-button></plastic-button>
```

The default generated render function code is (pseudo code):

```js
function render() {
  const component_plastic_button = resolveComponent("plastic-button")
  return createVNode(component_plastic_button)
}
```

And it will emit a warning if no component named `plastic-button` was found.

If the user wishes to use a native custom element named `plastic-button`, the desired generated code should be:

```js
function render() {
  return createVNode("plastic-button") // render as native element
}
```

To instruct the compiler to treat `<plastic-button>` as a custom element:

- If using a build step: pass the `isCustomElement` option to the Vue template compiler. If using `vue-loader`, this should be passed via `vue-loader`'s `compilerOptions` option:

  ```js
  // in webpack config
  rules: [
    {
      test: /\.vue$/,
      use: "vue-loader",
      options: {
        compilerOptions: {
          isCustomElement: tag => tag === "plastic-button"
        }
      }
    }
    // ...
  ]
  ```

- If using on-the-fly template compilation, pass it via `app.config`:

  ```js
  const app = Vue.createApp(/* ... */)

  app.config.isCustomElement = tag => tag === "plastic-button"
  ```

  Note the runtime config only affects runtime template compilation - it won't affect pre-compiled templates.

## Customized Built-in Elements

The Custom Elements specification provides a way to use custom elements as [Customized Built-in Element](https://html.spec.whatwg.org/multipage/custom-elements.html#custom-elements-customized-builtin-example) by adding the `is` attribute to a built-in element:

```html
<button is="plastic-button">Click Me!</button>
```

Vue's usage of the `is` special prop was simulating what the native attribute does before it was made universally available in browsers. However, in 2.x it is interpreted as rendering a Vue component with the name `plastic-button`. This blocks the native usage of Customized Built-in Element mentioned above.

In 3.0, we are limiting Vue's special treatment of the `is` prop to the `<component>` tag only.

- When used on the reserved `<component>` tag, it will behave exactly the same as in 2.x.

- When used on normal components, it will behave like a normal prop:

  ```html
  <foo is="bar" />
  ```

  - 2.x behavior: renders the `bar` component.
  - 3.x behavior: renders the `foo` component and passing the `is` prop.

- When used on plain elements, it will be passed to the `createElement` call as the `is` option, and also rendered as a native attribute. This supports the usage of customized built-in elements.

  ```html
  <button is="plastic-button">Click Me!</button>
  ```

  - 2.x behavior: renders the `plastic-button` component.
  - 3.x behavior: renders a native button by calling

    ```js
    document.createElement('button', { is: 'plastic-button' })
    ```

## `v-is` for In-DOM Template Parsing Workarounds

> Note: this section only affects cases where Vue templates are directly written in the page's HTML.

When using in-DOM templates, the template is subject to native HTML parsing rules. Some HTML elements, such as `<ul>`, `<ol>`, `<table>` and `<select>` have restrictions on what elements can appear inside them, and some elements such as `<li>`, `<tr>`, and `<option>` can only appear inside certain other elements.

In 2.x we recommended working around with these restrictions by using the `is` prop on a native tag:

``` html
<table>
  <tr is="blog-post-row"></tr>
</table>
```

With the behavior change of `is` proposed above, we need to introduce a new directive `v-is` for working around these cases:

``` html
<table>
  <tr v-is="'blog-post-row'"></tr>
</table>
```

Note `v-is` functions like a dynamic 2.x `:is` binding - so to render a component by its registered name, its value should be a JavaScript string literal.

# Adoption strategy

- Compat build can detect use of `config.ignoredElements` and provide appropriate warning and guidance.

- A codemod can automatically convert all 2.x non `<component>` tags with `is` usage to `<component is>` (for SFC templates) or `v-is` (for in-DOM templates).
