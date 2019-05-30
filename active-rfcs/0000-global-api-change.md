- Start Date: 2019-04-08
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Re-design app bootstrapping and global API.

- Global APIs that globally mutate Vue's behavior are now moved to **app instances** created the new `createApp` method, and their effects are now scoped to that app instance only.

- Global APIs that are do not mutate Vue's behavior (e.g. `nextTick` and the APIs proposed in [Advanced Reactivity API](https://github.com/vuejs/rfcs/pull/22)) are now named exports as specified in [the Global API Treeshaking RFC](https://github.com/vuejs/rfcs/blob/treeshaking/active-rfcs/0000-global-api-treeshaking.md).

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

new Vue({
  render: h => h(App)
}).$mount('#app')
```

## After

``` js
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)

app.config.ignoredElements = [/^app-/]
app.use(/* ... */)
app.mixin(/* ... */)
app.component(/* ... */)
app.directive(/* ... */)

app.mount('#app')
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
import App from './App.vue'

const app = createApp(App)
```

Calling `createApp` with a root component returns an **app instance**. An app instance provides an **app context**. The entire component tree formed by the root instance and its descendent components share the same app context, which provides the configurations that were previously "global" in Vue 2.x.

## Global API Mapping

An app instance exposes a subset of the current global APIs. The rule of thumb is any APIs that globally mutate Vue's behavior are now moved to the app instance. These include:

- Global configuration
  - `Vue.config` -> `app.config`
    - with the exception of `Vue.config.productionTip`
- Asset registration APIs
  - `Vue.component` -> `app.component`
  - `Vue.directive` -> `app.directive`
  - `Vue.filter` -> `app.filter`
- Behavior Extension APIs
  - `Vue.mixin` -> `app.mixin`
  - `Vue.use` -> `app.use`

All other global APIs that do not globally mutate behavior are now named exports as proposed in [Global API Treeshaking](https://github.com/vuejs/rfcs/pull/19).

## Mounting App Instance

The app instance can be mounted with the `mount` method. It works the same as the existing `vm.$mount()` component instance method and returns the mounted root component instance:

``` js
const rootInstance = app.mount('#app')

rootInstance instanceof Vue // true
```

The `mount` method can also accept props to be passed to the root component via the second argument:

``` js
app.mount('#app', {
  // props to be passed to root component
})
```

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

# Unresolved questions

- `Vue.config.productionTip` is left out because it is indeed "global". Maybe it should be moved to a global method?

  ``` js
  import { suppressProductionTip } from 'vue'

  suppressProductionTip()
  ```
