- Start Date: 2020-03-25
- Target Major Version: 3.x
- Reference Issues: N/A

# Summary

- **Breaking:** Custom Elements whitelisting is now performed during template compilation, and should be configured via compiler options instead of runtime config.
- **Breaking:** Restrict the special `is` prop usage to the reserved `<component>` tag only.

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
  const component_plastic_button = resolveComponent('plastic-button')
  return createVNode(component_plastic_button)
}
```

And it will emit a warning if no component named `plastic-button` was found.

If the user wishes to use a native custom element named `plastic-button`, the desired generated code should be:

```js
function render() {
  return createVNode('plastic-button') // render as native element
}
```

To instruct the compiler to treat `<plastic-button>` as a custom element:

- If using a build step: pass the `isCustomElement` option to the Vue template compiler. If using `vue-loader`, this should be passed via `vue-loader`'s `compilerOptions` option:

  ``` js
  // in webpack config
  rules: [
    {
      test: /\.vue$/,
      use: 'vue-loader',
      options: {
        compilerOptions: {
          isCustomElement: tag => tag === 'plastic-button'
        }
      }
    },
    // ...
  ]
  ```

- If using on-the-fly template compilation, pass it via `app.config`:

  ``` js
  const app = Vue.createApp(/* ... */)

  app.config.isCustomElement = tag => tag === 'plastic-button'
  ```

  Note the runtime config only affects runtime template compilation - it won't affect pre-compiled templates.

## Customized Built-in Elements

The Custom Elements specification provides a way to use custom elements as [Customized Built-in Element](https://html.spec.whatwg.org/multipage/custom-elements.html#custom-elements-customized-builtin-example) by adding the `is` attribute to a built-in element:

```html
<button is="plastic-button">Click Me!</button>
```

Vue's usage of the `is` special prop was simulating what the native attribute does before it was made universally available in browsers. However, in 2.x it is interpreted as rendering a Vue component with the name `plastic-button`. This blocks the native usage of Customized Built-in Element mentioned above.

In 3.0, we are restricting Vue's special treatment of the `is` prop to the reserved `<component>` tag only. So the example above would be rendering to the native `<button>` tag with the native `is` attribute.

# Adoption strategy

- Compat build can detect use of `config.ignoredElements` and provide appropriate warning and guidance.

- A codemod can automatically convert all 2.x tags with `is` usage to `<component>`.
