- Start Date: 2019-04-08
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Re-design app bootstrapping and global API.

- Global APIs that globally mutate Vue's behavior are now moved to **app instances** created by the new `createApp` method, and their effects are now scoped to that app instance only.

- Global APIs that are do not mutate Vue's behavior (e.g. `nextTick` and the APIs proposed in [Advanced Reactivity API](https://github.com/vuejs/rfcs/pull/22)) are now named exports as specified in [the Global API Treeshaking RFC](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0004-global-api-treeshaking.md).

# Basic example

## Before

``` js
import Vue from 'vue'
import App from './App.vue'

Vue.config.ignoredElements = [/^app-/]
Vue.use(/* ... */)
Vue.mixin(/* ... */)
Vue.component(/* ... */)
Vue.directive(/* ... */)

Vue.prototype.customProperty = () => {}

new Vue({
  render: h => h(App)
}).$mount('#app')
```

## After

``` js
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.config.isCustomElement = tag => tag.startsWith('app-')
app.use(/* ... */)
app.mixin(/* ... */)
app.component(/* ... */)
app.directive(/* ... */)

app.config.globalProperties.customProperty = () => {}

app.mount(App, '#app')
```

# Motivation

Some of Vue's current global API and configurations permanently mutate global state. This leads to a few problems:

- Global configuration makes it easy to accidentally pollute other test cases during testing. Users need to carefully store original global configuration and restore it after each test (e.g. resetting `Vue.config.errorHandler`). Some APIs (e.g. `Vue.use`, `Vue.mixin`) don't even have a way to revert their effects. This makes tests involving plugins particularly tricky.

  - `vue-test-utils` has to implement a special API `createLocalVue` to deal with this

- This also makes it difficult to share the same copy of `Vue` between multiple "apps" on the same page, but with different global configurations:

  ``` js
  // this affects both root instances
  Vue.mixin({ /* ... */ })

  const app1 = new Vue({ el: '#app-1' })
  const app2 = new Vue({ el: '#app-2' })
  ```

# Detailed design

Technically, Vue 2 doesn't have the concept of an "app". What we define as an app is simply a root Vue instance created via `new Vue()`. Every root instance created from the same `Vue` constructor shares the same global configuration.

In this proposal we introduce a new global API, `createApp`:

``` js
import { createApp } from 'vue'

const app = createApp({
  /* root component definition */
})
```

Calling `createApp` returns an **app instance**. An app instance provides an **app context**. The entire component tree mounted by the app instance share the same app context, which provides the configurations that were previously "global" in Vue 2.x.

## Global API Mapping

An app instance exposes a subset of the current global APIs. The rule of thumb is any APIs that globally mutate Vue's behavior are now moved to the app instance. These include:

- Global configuration
  - `Vue.config` -> `app.config`
    - `config.productionTip`: removed. ([details](#remove-configproductiontip))
    - `config.ignoredElements` -> `config.isCustomElement`. ([details](#configignoredelements---configiscustomelement))
    - `config.keyCodes`: removed. ([details](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0014-drop-keycode-support.md))
    - `config.optionMergeStrategies`: see [details](#configoptionmergestrategies-behavior-change) for behavior adjustments.
- Asset registration APIs
  - `Vue.component` -> `app.component`
  - `Vue.directive` -> `app.directive`
- Behavior Extension APIs
  - `Vue.mixin` -> `app.mixin`
  - `Vue.use` -> `app.use`

All other global APIs that do not globally mutate behavior are now named exports as proposed in [Global API Treeshaking](https://github.com/vuejs/rfcs/pull/19).

The only exception is `Vue.extend`. Since the global `Vue` is no longer a new-able constructor, `Vue.extend` no longer makes sense in terms of constructor extension.

- For extending a base component, the `extends` option should be used instead.
- For TypeScript type-inference, use the new `defineComponent` global API:

  ``` ts
  import { defineComponent} from 'vue'

  const App = defineComponent({
    /* Type inference provided */
  })
  ```

  Note that implementation-wise `defineComponent` does nothing - it simply returns the object passed to it. However, in terms of typing, the returned value has a synthetic type of a constructor for manual render function, TSX and IDE tooling support. This mismatch is an intentional trade-off.

## Mounting App Instance

The app instance can mount a root component with the `mount` method. It works similarly to the 2.x `vm.$mount()` method and returns the mounted root component instance:

``` js
const rootInstance = app.mount(App, '#app')
```

The `mount` method can also accept props to be passed to the root component via the third argument:

``` js
app.mount(App, '#app', {
  // props to be passed to root component
})
```

### Mounting Behavior Difference from 2.x

When using the compiler-included build and mounting a root component with no template of its own, Vue will attempt to use the mount target element's content as template. Note the differences between the 3.x behavior and 2.x:

- In 2.x, the root instance uses target element's `outerHTML` as template and replaces target element itself.

- In 3.x, the root instance uses target element's `innerHTML` as template and only replaces target element's children.

In most cases this should have no effect on how your app behaves, with the only side effect being that if the target element contains multiple children, the root instance will be mounted as a fragment and its `this.$el` will be pointing to the starting anchor node of the fragment (a DOM Comment node).

In Vue 3, due to the availability of Fragments, it is recommended to use template refs for direct access to DOM nodes instead of relying on `this.$el`.

## Provide / Inject

An app instance can also provide dependencies that can be injected by any component inside the app:

``` js
// in the entry
app.provide({
  [ThemeSymbol]: theme
})

// in a child component
export default {
  inject: {
    theme: {
      from: ThemeSymbol
    }
  },
  template: `<div :style="{ color: theme.textColor }" />`
}
```

This is similar to using the `provide` option in a 2.x root instance.

## Remove `config.productionTip`

In 3.0, the "use production build" tip will only show up when using the "dev + full build" (the build that includes the runtime compiler and has warnings).

For ES modules builds, since they are used with bundlers, and in most cases a CLI or boilerplate would have configured the production env properly, this tip will no longer show up.

## `config.ignoredElements` -> `config.isCustomElement`

This config option was introduced with the intention to support native custom elements, so the renaming better conveys what it does. The new option also expects a function which provides more flexibility than the old string / RegExp version:

``` js
// before
Vue.config.ignoredElements = ['my-el', /^ion-/]

// after
const app = Vue.createApp({ /* ... */ })
app.config.isCustomElement = tag => tag.startsWith('ion-')
```

**Important:** in 3.0, the check of whether an element is a component or not has been moved to the template compilation phase, therefore this config option is only respected when using the runtime compiler. If you are using the runtime-only build, `isCustomElement` must be passed to `@vue/compiler-dom` in the build setup instead - for example, via the [`compilerOptions` option in `vue-loader`](https://vue-loader.vuejs.org/options.html#compileroptions).

- If `config.isCustomElement` is assigned to when using a runtime-only build, a warning will be emitted instructing the user to pass the option in the build setup instead;

- This will be a new top-level option in the Vue CLI config.

## `config.optionMergeStrategies` Behavior Change

While still supported, due to Vue 3's internal implementation changes, built-in options no longer need merge strategies, so they are no longer exposed. The default `app.config.optionMergeStrategies` is now an empty object. This means:

- Users must now always provide their own merge strategy functions instead of reusing built-in strategies (e.g. you can no longer do `config.optionMergeStrategies.custom = config.optionMergeStrategies.props`)
- It is no longer possible to overwrite merge strategies for built-in options.

## Attaching Globally Shared Instance Properties

In 2.x, it was possible to inject globally shared instance properties by simply attaching them to `Vue.prototype`.

In Vue 3, since the global `Vue` is no longer a constructor, this is no longer supported. Instead, shared instance properties should be attached to an app instance's `config.globalProperties` instead:

``` js
// Before
Vue.prototype.$http = () => {}

// After
const app = createApp()
app.config.globalProperties.$http = () => {}
```

# Drawbacks

## Plugin auto installation

Many Vue 2.x libraries and plugins offer auto installation in their UMD builds, for example `vue-router`:

``` html
<script src="https://unpkg.com/vue"></script>
<script src="https://unpkg.com/vue-router"></script>
```

Auto installation relies on calling `Vue.use` which is no longer available. This should be a relatively easy migration, and we can expose a stub for `Vue.use` that emits a warning instead.

# Alternatives

N/A

# Adoption strategy

- The transformation is straightforward (as seen in the basic example).
- Moved methods can be replaced with stubs that emit warnings to guide migration.
- A codemod can also be provided.
- `config.ingoredElements` can be supported in the compat build.
- `config.optionMergeStrategies` built-in strategies can be supported in the compat build.
