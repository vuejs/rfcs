- Start Date: 19.04.2020
- Target Major Version: 2.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

VueTestUtils 2.x, which targets Vue 3, will introduce a few new methods and remove some less used ones.

- **Breaking:** `sync` mode removed. All `wrapper` methods return `nextTick`, you must `await` them
- **Breaking:** `find` is now split into `find` and `findComponent`.
- **Breaking:** Removal of some wrapper properties and methods, as they induce bad testing habits or are obsolete. 
- **Breaking:** `shallowMount` stubs defaults slots
- **Breaking:** `setProps` only works for the mounted component. 

**Note:** The API for VueTestUtils 1.x will stay the same and will support Vue 2.x. 

# Motivation

- Support Vue 3's Component APIs.
- Provide a smaller and easier to understand API
- Allow adding custom functionality via plugin system. [TODO]
- Remove methods that are bloat or lead to bad testing habits.
- Generally improve VTU Docs and Guides.

# Detailed design

- `createLocalVue` is removed. Vue now exposes `createApp`, which creates an isolated app instance. VTU does that under the hood.
- Fully `async`, each method that involves a mutation returns `nextTick`. Methods like `setValue` and `trigger` can be awaited, ensuring the DOM is re-rendered before each assertion.
- Rewritten completely in TypeScript, giving much improved type hints when writing tests.
- `shallowMount` will stub default slots of stubbed components. There will be an opt-in configuration to enable rendering slots for stubbed components in a similar manner to VTU beta. There is a limitation that scoped slots will not be able to provide data in such cases.
- Simple plugin system to allow extending VTU with your own methods. See [Plugins](#plugin-system)

## API changes

We will only list the changes and deprecations. Please check the temporary [documentation](https://vuejs.github.io/vue-test-utils-next-docs/) for a full API listing.

### mountOptions

#### props

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#props) - Renamed from `propsData` to match component `props` field.

#### global

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#global) - The `global` namespace is used to pass configuration to the `createApp` instance. Things like global components, directives and so on.

These settings can also be globally set via the exported `config` object - `config.global.mocks.$t = jest.fn()`.

- **global.components** - register global components
- **global.directives** - register a global directive
- **global.mixins** - register a global mixin
- **global.plugins** - install a plugin
- **global.stubs** - see bellow
- **global.mocks** - see bellow
- **global.provide** - see bellow

#### stubs 

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#global-stubs) - Moved to `global.stubs`. 

- **New** - Stubs will no longer render the slots of a component. This can be enabled by a global flag `config.renderStubSlots = true`.

#### mocks

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#global-mocks) - Moved to `global.mocks`

#### provide

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#global-provide) - Moved to `global.provide`.

### Methods

#### classes

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#classes)

- **New** - throw error for multiple root nodes.

#### unmount

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#unmount)

- **New** replaces `destroy` to match Vue 3 API

#### find

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#find)

- **Breaking** - Returns only `DOMWrapper`. Cannot find Component instances. see [findComponent](#findcomponent)
- **Breaking** - Accepts query selector only. 
- **New** - Can now return instance root element, or a fragment of the root.

#### findAll

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#findall)

- **Breaking** - Returns only array of `DOMWrapper`. 
- **Breaking** - No longer returns `WrapperArray`.
- **Breaking** - Accepts query selector only. 

#### findComponent

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#findcomponent)

`findComponent` can find component instances, deeply nested in your component tree. This is most useful for edge case assertions, that are not reflected in the DOM or when using `shallowMount` and asserting props on a stub.
 
Users should use `find` for the majority of cases, asserting the DOM properties, not the Vue components.

- **New** - finds a Vue Component instance by `ref`, `name`, `query` or Component definition. Returns `VueWrapper`.
- **New** - Only available on `VueWrapper`.

#### findAllComponents

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#findallcomponents)

- **New** - finds all Vue Components that match `name`, `query` or Component Definition. Returns array of `VueWrapper`.
- **New** - Only available on `VueWrapper`.

#### setProps

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#setprops)

- **Breaking** - Only works on the mounted component.
- **New** - Returns `nextTick`

#### setValue

[Link](https://vuejs.github.io/vue-test-utils-next-docs/api/#setvalue)

- **Breaking** - Only works on `DOMWrapper` (for now).
- **New** - Unifies `setChecked` and `setSelected`.
- **New** - Returns `nextTick`

### Plugin System

A simple plugin system is currently being discussed and prototyped here - [POC: VTU Plugin interface](https://github.com/vuejs/vue-test-utils-next/pull/82).

Thi should allow users to add extra methods to the VueWrapper and DOMWrapper classes, giving them more freedom in setting up their test suite.

```js
const plugin = (wrapper) => {
  return {
    width: 200,
    findByTestId: (query) => wrapper.find(`[data-testid=${query}]`)
  }
}

config.plugins.VueWrapper.install(plugin)

// later
expect(mount(Component).findByTestId('foo').exists()).toBe(true)
```

## Deprecated

### Methods

#### emittedByOrder

[Link](https://vue-test-utils.vuejs.org/api/wrapper/#emittedbyorder) - Rarely used, use `emitted` instead.

```js
expect(wrapper.emitted('change')[0]).toEqual(['param1', 'param2'])
```

#### is

[Link](https://vue-test-utils.vuejs.org/api/wrapper/#is) - Use `element.tagName` or the `classes()` method. Could be added as a plugin method later.

```js
expect(wrapper.element.tagName).toEqual('div')
expect(wrapper.classes()).toContain('Foo')
```

#### isEmpty

[Link](https://vue-test-utils.vuejs.org/api/wrapper/#isempty) - Use custom matcher like [jest-dom#tobeempty](https://github.com/testing-library/jest-dom#tobeempty) on the element.

```js
expect(wrapper.element).toBeEmpty()
```

#### isVisible

[Link](https://vue-test-utils.vuejs.org/api/wrapper/#isvisible) - Use custom matcher like [jest-dom#tobevisible](https://github.com/testing-library/jest-dom#tobevisible)

```js
expect(wrapper.element).toBeVisible()
```

#### isVueInstance

[Link](https://vue-test-utils.vuejs.org/api/wrapper/#isvueinstance) - No longer necessary, `find` always returns an `DOMWrapper` and `findComponent` returns a `VueWrapper`. Both return `ErrorWrapper` if failed.

#### setMethods

[Link](https://vue-test-utils.vuejs.org/api/wrapper/#setmethods) - Anti-pattern. Vue does not support arbitrarily replacement of methods, nor should VTU. If you need to stub out an action, extract the hard parts away. Then you can unit test them as well.

```js
// Component.vue
import { asyncAction } from 'actions'
const Component = {
    ...,
    methods: {
        async someAsyncMethod() {
            this.result = await asyncAction()
        }
    }	
}

// spec.js
import { asyncAction } from 'actions'
jest.mock('actions')
asyncAction.mockResolvedValue({ foo: 'bar' })

// rest of your test

```

#### setChecked and setSelected
 
Merged with [setValue](#setvalue)

#### destroy

Now named `unmount` to match Vue 3 API

#### name 

[Link](https://vue-test-utils.vuejs.org/api/wrapper/#name) - Removed from core. Could be added as part of extended plugin.

### Classes and properties

- **WrapperArray** - [Link](https://vue-test-utils.vuejs.org/api/wrapper-array/) - `find` and `findComponent` will just return an array of `VueWrapper` or `DOMWrapper` respectively.
- **config.methods** - [Link](https://vue-test-utils.vuejs.org/api/#methods) - Will no longer be able to replace methods.
- **config.silent** - [Link](https://vue-test-utils.vuejs.org/api/#silent) - not needed.
- **Wrapper.options** - [Link](https://vue-test-utils.vuejs.org/api/wrapper/#options) - not needed.

## Not yet implemented

- **Wrapper.selector** [Link](https://vue-test-utils.vuejs.org/api/wrapper/#selector)
- **Wrapper.contains** - [Link](https://vue-test-utils.vuejs.org/api/wrapper/#contains)
- **shallowMount** - [Link](https://vue-test-utils.vuejs.org/api/#shallowmount) - **Stubs** work, so its halfway there.
- **render** - [Link](https://vue-test-utils.vuejs.org/api/#render)
- **renderToString** - [Link](https://vue-test-utils.vuejs.org/api/#rendertostring)
- **createWrapper** - [Link](https://vue-test-utils.vuejs.org/api/#createwrapper-node-options)
- **enableAutoDestroy** - [Link](https://vue-test-utils.vuejs.org/api/#enableautodestroy-hook)
- **scopedSlots** - ScopedSlots are not ready yet. They will most probably be merged with normal ones, and will be a function with data, similar to VTU Beta.

# Drawbacks

- People will have to separate `find` into `findComponent` and `find`. We hope this would make tests easier to read and reason with.
- Snapshots would have to be updated.
- Some deprecated methods and functionality would have to be most likely installed via an extra plugin.

# Adoption strategy

- Rewrite docs from ground up. 
- Code-mod where possible (`find` -> `findComponent` for most cases, `destroy` -> `unmount`)
- Add deprecation warnings to beta before v1 release
- Add dedicated guides on how to write better and more maintainable tests for popular tools like Vuex, Router etc..
- Work with popular Vue ecosystem libraries and frameworks, like Quasar, Nuxt and Vuetify for better understanding of user needs.
- Deprecation build with warnings. 

# Unresolved questions

- Stubs is still in development. It has many issues in VTU beta and we want to do it right this time. Will probably post a new RFC entirely for it.
