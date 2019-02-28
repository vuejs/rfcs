- Start Date: 2019-02-26
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Introduce built-in support for authoring components as native ES2015 classes.

# Basic example

## In Browser

``` js
class App extends Vue {
  // options declared via static properties (stage 3)
  // more details below
  static template = `
    <div @click="increment">
      {{ count }} {{ plusOne }}
    </div>
  `

  // reactive data declared via class fields (stage 3)
  // more details below
  count = 0

  // lifecycle
  created() {
    console.log(this.count)
  }

  // getters are converted to computed properties
  get plusOne() {
    return this.count + 1
  }

  // a method
  increment() {
    this.count++
  }
}
```

The component will be mounted using a new global API instead of `new Vue()` - this will be discussed in a separate RFC.

## In Single File Components

``` html
<template>
  <div @click="increment">
    {{ count }} {{ plusOne }}
    <Foo />
  </div>
</template>

<script>
import Vue from 'vue'
import Foo from './Foo.vue'

export default class App extends Vue {
  static components = {
    Foo
  }

  count = 0

  created() {
    console.log(this.count)
  }

  get plusOne() {
    return this.count + 1
  }

  increment() {
    this.count++
  }
}
</script>
```

# Motivation

Vue's current object-based component API has created some challenges when it comes to type inference. As a result, most users opting into using Vue with TypeScript end up using [vue-class-component](https://github.com/vuejs/vue-class-component). This approach works, but with some drawbacks:

- Internally, Vue 2.x already represents each component instance with an underlying "class". We are using quotes here because it's not using the native ES2015 syntax but the ES5-style constructor/prototype function. Nevertheless, conceptually components are already handled as classes internally.

- `vue-class-component` had to implement some inefficient workarounds in order to provide the desired API without altering Vue internals.

- `vue-class-component` has to maintain typing compatibility with Vue core, and the maintenance overhead can be eliminated by exposing the class directly from Vue core.

The primary motivation of native class support is to provide a built-in and more efficient replacement for `vue-class-component`. The affected target audience are most likely also TypeScript users.

The API is also designed to not rely on anything TypeScript specific: it should work equally well in plain ES, for users who prefer using native ES classes.

**Note we are not pushing this as a replacement for the existing object-based API - the object-based API will continue to work in 3.0.**

# Detailed design

## Basics

A component can be declared by extending the base `Vue` class provided by Vue core:

``` js
import Vue from 'vue'

class MyComponent extends Vue {}
```

### Data

Reactive instance data properties can be declared using [class fields syntax (stage 3)](https://github.com/tc39/proposal-class-fields):

``` js
class MyComponent extends Vue {
  count = 0
}
```

This is currently supported in Chrome stable 72+ and TypeScript. It can also be transpiled using Babel. If using native ES classes without any transpilation, it's also possible to manually set `this.count = 0` in `constructor`, which would in turn require a `super()` call:

``` js
// NOT recommended.
class MyComponent extends Vue {
  constructor() {
    super()
    this.count = 0
  }
}
```

This is verbose and also has incorrect semantics (see below). A less verbose alternative is using the special `data()` method, which works the same as in the object-based syntax:

``` js
class MyComponent extends Vue {
  data() {
    return {
      count: 0
    }
  }
}
```

#### A Note on `[[Set]]` vs `[[Define]]`

The class field syntax uses `[[Define]]` semantics in both native and transpiled implementations (Babel already conforms to the latest spec and TS will have to follow suite). This means `count = 0` in the class body is executed with the semantics of `Object.defineProperty` and will always overwrite a property of the same name inherited from a parent class, regardless of whether it has a setter or not.

In comparison, `this.count = 0` in constructor is using `[[Set]]` semantics - if the parent class has a defined setter named `count`, the operation will trigger the setter instead of overwriting the definition.

For Vue's API, `[[Define]]` is the correct semantics, since an extended class declaring a data property should overwrite a property with the same name on the parent class.

This should be a very rare edge case since most users will likely be using the class field syntax either natively or via a transpiler with correct semantics, or using the `data()` alternative.

### Lifecycle Hooks

Built-in lifecycle hooks should be declared directly as methods, and works largely the same with their object-based counterparts:

``` js
class MyComponent extends Vue {
  created() {
    console.log('created')
  }
}
```

### Props

In v3, props declarations can be optional. The behavior will be different based on whether props are declared.

#### Props with Explicit Declaration

Props can be declared using the `props` static property (static properties are used for all component options that do not have implicit mapping). When props are declared, they can be accessed directly on `this`:

``` js
class MyComponent extends Vue {
  // props declarations are fully compatible with v2 options
  static props = {
    msg: String
  }

  created() {
    // available on `this`
    console.log(this.msg)

    // also available on `this.$props`
    console.log(this.$props.msg)
  }
}
```

Similar to v2, any attributes passed to the component but is not declared as a prop will be exposed as `this.$attrs`. Note that the non-props attribute fallthrough behavior will also be adjusted - [it is discussed in more details in a separate RFC](TODO).

#### Props without Explicit Declaration

It is possible to omit props declarations in v3. When there is no explicit props declaration, props will **NOT** be exposed on `this` - they will only be available on `this.$props`:

``` js
class MyComponent extends Vue {
  created() {
    console.log(this.$props.msg)
  }
}
```

Inside templates, the prop also must be accessed with the `$props` prefix, .e.g. `{{ $props.msg }}`.

Any attribute passed to this component will be exposed in `this.$props`. In addition, `this.$attrs` will be simply pointing to `this.$props` since they are equivalent in this case.

### Computed Properties

Computed properties are declared as getter methods:

``` js
class MyComponent extends Vue {
  count = 0

  get doubleCount() {
    return this.count * 2
  }
}
```

Note although we are using the getter syntax, these functions are not used a literal getters - they are converted into Vue computed properties internally with dependency-tracking-based caching.

> Do we need a way to opt-out? It can probably be done via decorators.

### Methods

Any method that is not a reserved lifecycle hook is considered a normal instance method:

``` js
class MyComponent extends Vue {
  count = 0

  created() {
    this.logCount()
  }

  logCount() {
    console.log(this.count)
  }
}
```

When methods are accessed from `this`, they are **automatically bound to the instance.** This means there is no need to worry about calling `this.foo = this.foo.bind(this)`.

### Other Options

Other options that do not have implicit mapping in the class syntax should be declared as static class properties:

``` js
class MyComponent extends Vue {
  static template = `
    <div>hello</div>
  `
}
```

The above syntax requires [static class fields (stage 3)](https://github.com/tc39/proposal-static-class-features). In non-supporting environment, manual attaching is required:

``` js
class MyComponent extends Vue {}

MyComponent.template = `
  <div>hello</div>
`
```

Or:

``` js
class MyComponent extends Vue {}

Object.assign(MyComponent, {
  template: `
    <div>hello</div>
  `
})
```

## TypeScript Usage

In TypeScript, since `data` properties are declared using class fields, the type inference just works:

``` ts
class MyComponent extends Vue {
  count: number = 1

  created() {
    this.count // number
  }
}
```

For props, we intend to provide a decorator that internally transforms decorated fields in to corresponding runtime options (similar to [the `@Prop` decorator in `vue-property-decorators`](https://github.com/kaorun343/vue-property-decorator#Prop)):

``` ts
import { prop } from 'vue'

class MyComponent extends Vue {
  @prop count: number

  created() {
    this.count // number
  }
}
```

This is equivalent to the following in terms of runtime behavior (only static type checking, no runtime checks):

``` ts
class MyComponent extends Vue {
  static props = ['count']

  created() {
    this.count
  }
}
```

The decorator can also be called with additional options for more specific runtime behavior:

``` ts
import { prop } from 'vue'

class MyComponent extends Vue {
  @prop({
    validator: val => {
      // custom runtime validation logic
    }
  })
  msg: string = 'hello'

  created() {
    this.count // number
  }
}
```

### Note on Prop Default Value

Note that due to the limitations of the TypeScript decorator implementation, we cannot use the following to declare default value for a prop:

``` ts
class MyComponent extends Vue {
  @prop count: number = 1
}
```

The culprit is the following case:

``` ts
class MyComponent extends Vue {
  @prop foo: number = 1
  bar = this.foo + 1
}
```

If the parent component passes in the `foo` prop, the default value of `1` should be overwritten. However, the way TypeScript transpiles the code places the two lines together in the constructor of the class, giving Vue no chance to overwrite the default value properly.

Instead, use the decorator option to declare default values:

``` ts
class MyComponent extends Vue {
  @prop({ default: 1 }) foo: number
  bar = this.foo + 1
}
```

This restriction can be enforced through lint rules. It can also be lifted in the future when the ES decorators proposal has been finalized and TS has been updated to match the spec (or pass-through to native decorators), assuming the final spec does not deviate too much from how it works now.

### `$props` and `$data`

To access `this.$props` or `this.$data` in TypeScript, the base `Vue` class accepts generic arguments:

``` ts
interface MyProps {
  msg: string
}

interface MyData {
  count: number
}

class MyComponent extends Vue<MyProps, MyData> {
  count: number = 1

  created() {
    this.$props.msg
    this.$data.count
  }
}
```

## Mixins

Mixins work a bit differently with classes, primarily to ensure proper type inference:

1. If type inference is needed, mixins must be declared as classes extending the base `Vue` class (otherwise, the object format also works).

2. To use mixins, the final component should extend a class created from the `mixins` method instead of the base `Vue` class.

``` ts
import Vue, { mixins } from 'vue'

class MixinA extends Vue {
  // class-style mixin
}

const MixinB = {
  // object-style mixin
}

class MyComponent extends mixins(MixinA, MixinB) {
  // ...
}
```

The class returned from `mixins` also accepts the same generics arguments as the base `Vue` class.

## Difference from 2.x Constructors

One major difference between 3.0 classes and the 2.x constructors is that they are not meant to be instantiated directly. i.e. you will no longer be able to do `new MyComponent({ el: '#app' })` to mount it - instead, the instantiation/mounting process will be handled by separate, dedicated APIs. In cases where a component needs to be instantiated for testing purposes, corresponding APIs will also be provided. This is largely due to the internal changes where we are moving the mounting logic out of the component class itself for better decoupling, and also has to do our plan to redesign the global API for bootstrapping an app.

# Drawbacks

## Reliance on Stage 2/3 Language Features

### Class Fields

The proposed syntax relies on two currently stage-3 proposals related to class fields:

- [Class fields](https://github.com/tc39/proposal-class-fields)
- [Static class features](https://github.com/tc39/proposal-static-class-features)

These are required to achieve the ideal usage. Although there are workarounds in cases where they are not available, the workarounds result in sub-optimal authoring experience.

If the user uses Babel or TypeScript, these can be covered. Luckily these two combined should cover a pretty decent percentage of all users. For learning / prototyping usage without compile steps, browsers with native support (e.g. Chrome Canary) can also be used.

There is a small risk since these proposals are just stage 3, and are still being actively debated on - technically, there are still chances that they get further revised or even dropped. The good news is that the parts that are relevant here doesn't seem likely to change. There was a somewhat related debate regarding the semantics of class fields being `[[Set]]` vs `[[Define]]`, and it has been settled as `[[Define]]` which in my opinion is the preferred semantics for this API.

### Decorators

The TypeScript usage relies on decorators. The [decorators proposal](https://github.com/tc39/proposal-decorators) for JavaScript is still stage 2 and undergoing major revisions - it's also completely different from how it is implemented in TS today (although TS is expected to match the proposal once it is finalized). Its latest form just got rejected from advancing to stage 3 at TC39 due to concerns from JavaScript engine implementors. It is thus still quite risky to design the API around decorators at this point.

**Before ES decorators are finalized, we only recommend using decorators in TypeScript.**

The decision to go with decorators for props in TypeScript is due to the following:

1. Decorators is the only option that allows us to express both static and runtime behavior in the same syntax, without the need for double declaration. This is discussed in more details in the [Alternatives](#declaring-prop-types-via-generic-arguments) section.

2. Both the current TS implementation and the current stage 2 proposal can support the desired usage.

3. It's also highly likely that the finalized proposal is going to support the usage as well. So even after the proposal finalizes and TS' implementation has been updated to match the proposal, the API can continue to work without syntax changes.

4. The decorator-based usage is opt-in and built on top of the `static props` based usage. So even if the proposal changes drastically or gets abandoned we still have something to fallback to.

5. If users are using TypeScript, they already have decorators available to them via TypeScript's tool chain so unlike vanilla JavaScript there's no need for additional tooling.

## `this` Identity in `constructor`

In Vue 3 component classes, the `this` context in all lifecycle hooks and methods are in fact a Proxy to the actual underlying instance. This Proxy is responsible for returning proper values for the data, props and computed properties defined on the current component, and provides runtime warning checks. It is important for performance reasons as it avoids many expensive `Object.defineProperty` calls when instantiating components.

In practice, your code will work exactly the same - the only cases where you need to pay attention is if you are using `this` inside the native `constructor` - this is the only place where Vue cannot swap the identity of `this` so it will not be equal to the `this` exposed everywhere else:

``` js
let instance

class MyComponent extends Vue {
  constructor() {
    super()
    instance = this // actual instance
  }

  created() {
    console.log(this === instance) // false, `this` here is the Proxy
  }
}
```

In practice, there shouldn't be cases where you must use the `constructor`, so the best practice is to simply avoid it and always use component lifecycle hooks.

## Two Ways of Doing the Same Thing

This may cause beginners to face a choice early on: to go with the object syntax, or the class syntax?

For users who already have a preference, it is not really an issue. The real issue is that for beginners who are not familiar with classes, the syntax raises the learning barrier. In the long run, as ES classes stabilize and get more widely used, it may eventually become a basic pre-requisite for all JavaScript users, but now is probably not the time yet.

One way to deal with it is providing examples for both syntaxes in the new docs and allow switching between them. This allows users to pick a preferred syntax during the learning process.

## Isn't React Moving Away from Classes?

Yes, but that's because classes isn't as nicely a fit for React's component conceptual model, especially with hooks being the alternative. Classes aren't inherently bad, and Vue's component conceptual model (with mutable state) maps better to a class than a React component. We also have plans to offer a mechanism with hooks-like logic composition capabilities that works in both class and object-based components, but that will be in a separate future RFC.

# Alternatives

## Options via Decorator

``` js
@Component({
  template: `...`
})
class MyComponent extends Vue {}
```

This is similar to `vue-class-component` but it requires decorators - and as mentioned, it is only stage 2 and risky to rely on. We are using decorators for props, but it's primarily for better type-inference and only recommended in TypeScript. For now we should avoid decorators in plain ES as much as possible.

## Declaring Prop Types via Generic Arguments

For declaring prop types in TypeScript, we considered avoiding decorators by merging the props interface passed to the class as a generic argument on to the class instance:

``` ts
interface MyProps {
  msg: string
}

class MyComponent extends Vue<MyProps> {
  created() {
    this.msg // this becomes available
  }
}
```

However, this creates a mismatch between the typing and the runtime behavior. Because there is no runtime declaration for the `msg` prop, it will **not** be exposed on `this`. To make the types and runtime consistent, we end up with a double-declaration:

``` ts
interface MyProps {
  msg: string
}

class MyComponent extends Vue<MyProps> {
  static props = ['msg']

  created() {
    this.msg
  }
}
```

We also considered eliminating the need for double-declaration via tooling - e.g. [Vetur](https://github.com/vuejs/vetur) can pre-transform the interface into equivalent runtime declaration, or vice-versa, so that only the interface or the `static props` declaration is needed. However, both have drawbacks:

- The interface cannot enforce runtime type checking or custom validation;

- The `static props` runtime declaration cannot facilitate type inference for advanced type shapes.

Decorators is the only option the can unify both in the same syntax:

``` ts
class MyComponent extends Vue {
  @prop({
    validator: value => {
      // custom runtime validation logic
    }
  })
  msg: SomeAdvancedType = 'hello'

  created() {
    this.msg
  }
}
```

# Adoption strategy

- This does not break existing usage, but rather introduces an alternative way of authoring components. TypeScript users, especially those already using `vue-class-component` should have no issue grasping it. For beginners, we should probably avoid using it as the default syntax in docs, but we should provide the option to switching to it in code examples.

- For existing users using TypeScript and `vue-class-component`, a simple migration strategy would be shipping a build of `vue-class-component` that maintains the same API, but with greatly simplified implementation. The `@Component` decorator would simply spread the options on to the native class as static properties. Then the user can migrate away from the decorator to static properties. Because the required change is pretty mechanical, a code mod can also be provided.
