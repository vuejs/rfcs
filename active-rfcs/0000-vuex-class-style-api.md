- Start Date: 2019-02-24
- Target Major Version: Vuex 3.x
- Reference Issues:
  - https://github.com/vuejs/vuex/issues/532
  - https://github.com/vuejs/vuex/issues/564
- Implementation PR:

# Summary

<!-- Brief explanation of the feature. -->

Supporting class-style API for Vuex module so that we write a module in type safe mannar in TypeScript.

# Basic example

<!-- If the proposal involves a new or changed API, include a basic code example.
Omit this section if it's not applicable. -->

The below is basic example to write a module.

```js
import { Store, Getters, Mutations, Actions } from 'vuex'

/**
 * State
 */
class CounterState {
  count = 0
}

/**
 * Getters
 */
class CounterGetters extends Getters {
  /**
   * Getter as getter property (will be cached by Vue's reactive system)
   */
  get double() {
    return this.state.count * 2
  }

  /**
   * Getter with argument
   */
  sum(n) {
    return this.state.count + n
  }

  /**
   * Using other getter
   */
  get triple() {
    return this.state.count + this.getters.double
  }
}

/**
 * Mutations
 */
class CounterMutations extends Mutations {
  /**
   * The first argument is a payload.
   */
  increment(payload) {
    this.state.count + payload.value
  }
}

/**
 * Actions
 */
class CounterActions extends Actions {
  /**
   * The first argument is a payload.
   * Each value of action context is under `this`.
   */
  increment(payload) {
    this.commit('increment', payload)
  }
}

const store = new Store({
  state: CounterState,
  getters: CounterGetters,
  mutations: CounterMutations,
  actions: CounterActions
})
```

You can retain some external object in actions which is not suitable to be in state (e.g. WebSocket connection).

```js
import { Actions } from 'vuex'

class WsActions extends Actions {
  // Dispatch init action for creating WebSocket connection.
  init() {
    this.ws = new WebSocket('wss://...')
  }

  // Use initialized connection on the other actions.
  emit() {
    this.ws.send('...')
  }
}
```

In TypeScript, we pass some type parameters for base class so that the module will be typed.

```ts
import { Getters, Mutations, Actions } from 'vuex'

/**
 * State
 */
class CounterState {
  // ...
}

/**
 * Getters
 * Pass module state type
 */
class CounterGetters extends Getters<CounterState> {
  // ...
}

/**
 * Mutations
 * Pass module state type
 */
class CounterMutations extends Mutations<CounterState> {
  // ...
}

/**
 * Actions
 * Pass module state, getters, mutation and action itself types
 * (Specifying action type is needed in terms of implementation)
 */
class CounterActions extends Actions<
  CounterState,
  CounterGetters,
  CounterMutations,
  CounterActions
> {
  // ...
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

The main motivation is type safety. The current object-style API is hard to infer
state, getters, actions and mutations types in a module because they are circularly referenced by each other.

There is [an attempt](https://github.com/vuejs/vuex/pull/1121) to type with the current API but it is tricky –
It requires separated type declarations for module assets and manually annotating them to a module implementation.
It could be also verbose as we need to write action name both on action type and its implementation, for example.

The class-style API itself provides type-safety **in** modules themselves which means all usage for local state, getters,
actions and mutations are strictly typed. However it is not typed that accessing a module asset from a component or
another module yet. This will covered by another proposal that is using module type information which class-style syntax provides.
[TODO: refer to the proposal]

The class-style API also solves the case that we need to manage some external object in actions which is not suitable
to be in state. In the current object-style API, we need to put a variable to retain such object out of actions object
but the variable is actually shared if the actions are reused on the multiple places in a store.

```js
// The current work around for managing an external object in actions.

// Declare varable to retain WebSocket connection.
// But this variable is a shared object.
let ws

const actions = {
  init() {
    ws = new WebSocket('wss://...')
  },

  emit() {
    ws.send(/* ... */)
  }
}
```

# Detailed design

<!--
This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with Vue to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.
-->

Vuex provides the following classes to create module:

- `Getters`
- `Mutations`
- `Actions`

They should be imported by ES import statement:

```js
import { Getters, Mutations, Actions } from 'vuex'
```

## Module state

Module state can be defined in class-style syntax:

```ts
class CounterState {
  count = 0
}
```

By class-style syntax, the user defines both type and value of state object in one time.
That means the `CounterState` in the above example can be passed as state value while
used for other module assets as type.

## Getters base class

`Getters` class has following properties:

- `state`: module local state.
- `getters`: module local getters.
- `rootState`: root state.
- `rootGetters`: root getters.

The user creates module getters in the extended class of `Getters`. The `Getters` class receives one type parameter
to specify the local state type in TypeScript:

```ts
class CounterGetters extends Getters<CounterState> {
  // Define `double` getter
  get double() {
    // `this.state` is typed with `CounterState`
    return this.state.count * 2
  }
}
```

If the getters are defined as getter property (`get getterName()`), they will be registered as computed properties under the hood
(the same behaviour of the current getter). If they are method, they can receive arguments.

```ts
class CounterGetters extends Getters<CounterState> {
  get computed() {
    /* ... */
  }

  method() {
    /* ... */
  }

  methodWithArgs(arg) {
    /* ... */
  }
}

const store = new Store(/* ... */)

// computed property getter
store.getters.computed

// method getter
store.getters.method()

// method getter with argument
store.getters.methodWithArgs(123)
```

## Mutations base class

`Mutations` class has `state` property and the user updates it in each mutation. The user defines mutations as method in
an extended class of `Mutations`. The mutation methods can receive 1st argument as mutation payload.
The `Mutations` class receives one type parameter to specify the local state type in TypeScript:

```ts
class CounterMutations extends Mutations<CounterState> {
  increment(payload) {
    // `this.state` is typed with `CounterState`
    this.state.count += payload.amount
  }
}
```

## Actions base class

`Actions` class has the following properties:

- `state`: module local state.
- `getters`: module local getters.
- `rootState`: root state.
- `rootGetters`: root getters.
- `dispatch`: method to call local action. Can call root action with `{ root: true }`
- `commit`: method to call local mutation. Can call root mutation with `{ root: true }`

The user defines actions as method in an extended class of `Actions`. The action methods can receive 1st argument as
action payload. The `Actions` class receives four type parameter to specify local state, getters, mutations and
actions type in this order in TypeScript:

```ts
class CounterActions extends Actions<
  CounterState,
  CounterGetters,
  CounterMutations,
  CounterActions
> {
  increment(payload) {
    this.commit('increment', payload)
  }
}
```

If the user wants to omit some type parameters, they can use `any` or `never` for them.

```ts
// For the module without getters
class CounterActions extends Actions<
  CounterState,
  never,
  CounterMutations,
  CounterActions
> {
  // ...
}
```

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

## Can be implemented as a wrapper

The proposed API can be implmented as a third-party wrapper library (e.g. https://github.com/ktsn/vuex-smart-module).
But it would be worth adding to the core because the need of type safety in Vuex is very common in Vuex+TypeScript people.

## Possible abuse of `this`

Since the user can access other method and properties via `this`, there is a concern that it is abused.
For example, they can technically call some mutation in another mutation which is forbidden in Vuex:

```js
class CounterMutations extends Mutations {
  increment() {
    this.state.count++
  }

  doubleIncrement() {
    // SHOULD NOT do this: calling another mutation
    this.increment()
    this.increment()
  }
}
```

The same goes for getters and actions.

We can set runtime check to prohibit such kind of usage in the development build though.

# Alternatives

<!--
What other designs have been considered? What is the impact of not doing this?
-->

- Creating modules by using ES decorator like [vuex-module-decorators](https://github.com/championswimmer/vuex-module-decorators).

# Adoption strategy

<!--
If we implement this proposal, how will existing Vue developers adopt it? Is
this a breaking change? Can we write a codemod? Can we provide a runtime adapter library for the original API it replaces? How will this affect other projects in the Vue ecosystem?
-->

The class-style API does not break existing usage. If the users want to introduce it in some existing apps,
they can incrementally replace it with the existing syntax.

# Unresolved questions

<!--
Optional, but suggested for first drafts. What parts of the design are still
TBD?
-->

- Should we port action / getter context via another object so that we can avoid possible conflict
  when we add a new property to context? (like `this.context.state`)
