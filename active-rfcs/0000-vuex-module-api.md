- Start Date: 2019-02-24
- Target Major Version: Vuex 3.x
- Reference Issues: https://github.com/vuejs/vuex/issues/564
- Implementation PR:

# Summary

<!-- Brief explanation of the feature. -->

Introducing a way to access module assets via a module object instead of namespace string.

The module object has:

- component mapper methods (`mapXXX`) for corresponding module assets.
- module local context generator to access module assets out of components.

They will be strictly typed by utilizing module object type in TypeScript.

This proposal is based on [class-style API proposal](https://github.com/vuejs/rfcs/pull/14) because it needs that
the module assets are strictly typed which can be achieved the class-style API.

# Basic example

```js
import { Store, Module } from 'vuex'

// create module object
export const counter = new Module({
  // pass module options here
  state: {
    /* ... */
  }
})

export const store = new Store(counter, {
  strict: true
})
```

In components:

```js
import Vue from 'vue'
import { counter } from '@/store/modules/counter'

export default Vue.extends({
  // module objects have mapXXX helpers

  computed: counter.mapGetters(['...']),

  methods: counter.mapActions(['...'])
})
```

In another module:

```js
import { Actions } from 'vuex'
import { counter } from '@/store/modules/counter'

// class-style syntax
class AnotherActions extends Actions {
  // create module context of counter module
  counter = counter.context(this.store)

  incrementCounter() {
    // commit `increment` mutation in counter
    this.counter.commit('increment')
  }
}

// object-style syntax
const anotherActions = {
  incrementCounter() {
    const counterCtx = counter.context(this)
    counterCtx.commit('increment')
  }
}
```

# Motivation

<!--
Why are we doing this? What use cases does it support? What is the expected
outcome?

Please focus on explaining the motivation so that if this RFC is not accepted,
the motivation could be used to develop alternative solutions. In other words,
enumerate the constraints you are trying to solve without coupling them too
closely to the solution you have in mind.
-->

To ensure type safety when referring module assets from components or any other place in apps.
Currently, it is not typed at all because:

1. `mapXXX` helpers cannot know the type of module assets,
2. It is impossible to deal with namespace string on type level and
3. typing store instance is too complicated as it has nested structure and needs strinc concatnation on type level.

To solve these problem, we utilize a module object which is fully typed by class-style syntax. And automatically resolve
namespace string in the module object.

# Detailed design

<!--
This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with Vue to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.
-->

## Module object

Providing `Module` constructor to create a module object. It receives module options which is the same shape with the current plain object module:

```js
import { Module } from 'vuex'

// create counter module
export const counter = new Module({
  namespaced: true,

  // module assets
  state: ...,
  getters: ...,
  mutations: ...,
  actions: ...,

  // nested modules
  modules: {
    ...
  }
})
```

The module object can be passed to the store constructore's first argument. If the first argument is the module object, the second argument will be store options:

```js
import { Store } from 'vuex'
import { counter } from './modules/counter'

export default new Store(counter, {
  strict: true
})
```

After registering a module object, it will store the registered module path and namespace string under the hood so that the user do not have to specify it by their hand.
The module object has to be unique in a store because of this behaviour.

If the user wants to reuse a module in serveral places in a store, they need to clone it by using `clone` method:

```js
import { Store } from 'vuex'
import { counter } from './modules/counter'

// clone counter module
const anotherCounter = counter.clone()

export default new Store({
  modules: {
    counter,
    anotherCounter
  }
})
```

## Component mappers on module object

The module object has following component mappers as method:

- `mapState`
- `mapGetters`
- `mapMutations`
- `mapActions`

They has the same interface with the existing `mapXXX` helpers except they do not accept namespace string.

```js
import Vue from 'vue'
import { counter } from '@/store/modules/counter'

export default Vue.extend({
  computed: {
    ...counter.mapGetters(['a']),

    ...counter.mapState({
      b: 'c',

      d: (state, getters) => {
        return state.d
      }
    })
  },

  methods: {
    ...counter.mapMutations(['e']),

    ...counter.mapActions({
      f: 'g',

      h: (dispatch, n) => {
        return dispatch('h', n)
      }
    })
  }
})
```

## Module context

The user can generate module context object by passing a store instance to `context()` method.
The context object allow to access corresponding module assets like action context:

```js
const ctx = counter.context(store)

ctx.state
ctx.getters
ctx.dispatch('increment')
ctx.commit('increment')
```

This can be used in router hooks, to cooperate multiple module and so on in type safe way:

```js
import { Getters, Actions } from 'vuex'
import { counter } from '@/store/modules/counter'

class AnotherGetters extends Getters {
  // create counter context
  counter = counter.context(this.store)

  get counterValue() {
    // use counter state
    return this.counter.state.count
  }
}

class AnotherActions extends Actions {
  // create counter context
  counter = counter.context(this.store)

  incrementCounter() {
    // dispatch counter mutation
    this.counter.dispatch('increment')
  }
}
```

To use the module context in module, we need to add store reference in getters and actions.
We can add `store` property to `Getters` and `Actions` base class and make `this` in object-style getters to be store.

# Drawbacks

<!--
Why should we _not_ do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people Vue
- integration of this feature with other existing and planned features
- cost of migrating existing Vue applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.
-->

- This can be implemented in user land (https://github.com/ktsn/vuex-smart-module). But this level of type safety would be needed by default.

# Alternatives

<!--
What other designs have been considered? What is the impact of not doing this?
-->

- Creating accessor function for each module asset instead of a module like [vuex-typescript](https://github.com/istrib/vuex-typescript/).

# Adoption strategy

<!--
If we implement this proposal, how will existing Vue developers adopt it? Is
this a breaking change? Can we write a codemod? Can we provide a runtime adapter library for the original API it replaces? How will this affect other projects in the Vue ecosystem?
-->

This proposal is fully backward compatible as only adding a new API.

# Unresolved questions

N/A
