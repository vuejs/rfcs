- Start Date: 2019-05-30
- Target Major Version: 2.x / 3.x
- Reference Issues:
- Implementation PR: (leave this empty)

# Summary

Expose logic-related component options via function-based APIs instead.

# Basic example

``` js
import { value, computed, watch, onMounted } from 'vue'

const App = {
  template: `
    <div>
      <span>count is {{ count }}</span>
      <span>plusOne is {{ plusOne }}</span>
      <button @click="increment">count++</button>
    </div>
  `,
  setup() {
    // reactive state
    const count = value(0)
    // computed state
    const plusOne = computed(() => count.value + 1)
    // method
    const increment = () => { count.value++ }
    // watch
    watch(() => count.value * 2, val => {
      console.log(`count * 2 is ${val}`)
    })
    // lifecycle
    onMounted(() => {
      console.log(`mounted`)
    })
    // expose bindings on render context
    return {
      count,
      plusOne,
      increment
    }
  }
}
```

# Motivation

## Logic Composition

One of the key aspects of the component API is how to encapsulate and reuse logic across multiple components. With Vue 2.x's current API, there are a number of common patterns we've seen in the past, each with its own drawbacks. These include:

- Mixins (via the `mixins` option)
- Higher-order components (HOCs)
- Renderless components (via scoped slots)

These patterns are discussed in more details in the [appendix](#prior-art-composition-patterns) - but in general, they all suffer from one or more of the drawbacks below:

- Unclear sources for properties exposed on the render context. For example, when reading the template of a component using multiple mixins, it can be difficult to tell from which mixin a specific property was injected from.

- Namespace clashing. Mixins can potentially clash on property and method names, while HOCs can clash on expected prop names.

- Performance. HOCs and renderless components require extra stateful component instances that come at a performance cost.

The function based API, inspired by [React Hooks](https://reactjs.org/docs/hooks-intro.html), presents a clean and flexible way to compose logic inside and between components without any of these drawbacks. This can be achieved by extracting code related to a piece of logic into what we call a "composition function" and returning reactive state. Here is an example of using a composition function to extract the logic of listening to the mouse position:

``` js
function useMouse() {
  const x = value(0)
  const y = value(0)
  const update = e => {
    x.value = e.pageX
    y.value = e.pageY
  }
  onMounted(() => {
    window.addEventListener('mousemove', update)
  })
  onUnmounted(() => {
    window.removeEventListener('mousemove', update)
  })
  return { x, y }
}

// in consuming component
const Component = {
  setup() {
    const { x, y } = useMouse()
    const { z } = useOtherLogic()
    return { x, y, z }
  },
  template: `<div>{{ x }} {{ y }} {{ z }}</div>`
}
```

Note in the example above:

- Properties exposed to the template have clear sources since they are values returned from composition functions;
- Returned values from composition functions can be arbitrarily named so there is no namespace collision;
- There are no unnecessary component instances created just for logic reuse purposes.

See also:

- [Appendix: Comparison with React Hooks](#comparison-with-react-hooks)

## Type Inference

One of the major goals of 3.0 is to provide better built-in TypeScript type inference support. Originally we tried to address this problem with the now-abandoned [Class API RFC](https://github.com/vuejs/rfcs/pull/17), but after discussion and prototyping we discovered that using Classes [doesn't fully address the typing issue](#type-issues-with-class-api).

The function-based APIs, on the other hand, are naturally type-friendly. In the prototype we have already achieved full typing support for the proposed APIs.

See also:

- [Appendix: Type Issues with Class API](#type-issues-with-class-api)

## Bundle Size

Function-based APIs are exposed as named ES exports and imported on demand. This makes them tree-shakable, and leaves more room for future API additions. Code written with function-based APIs also compresses better than object-or-class-based code, since (with standard minification) function and variable names can be shortened while object/class methods and properties cannot.

# Detailed design

## The `setup` function

A new component option, `setup()` is introduced. As the name suggests, this is the place where we use the function-based APIs to setup the logic of our component. `setup()` is called when an instance of the component is created, after props resolution. The function receives the resolved props as its argument:

``` js
const MyComponent = {
  props: {
    name: String
  },
  setup(props) {
    console.log(props.name)
  }
}
```

Note this `props` object is reactive - i.e. it is updated when new props are passed in, and can be observed and reacted upon using the `watch` function introduced later in this RFC. However, for userland code, it is immutable during development (will emit warning if user code attempts to mutate it).

`this` is usable inside `setup()`, but you most likely won't need it very often.

## State

Similar to `data()`, `setup()` can return an object containing properties to be exposed to the template's render context:

``` js
const MyComponent = {
  props: {
    name: String
  },
  setup(props) {
    return {
      msg: `hello ${props.name}!`
    }
  },
  template: `<div>{{ msg }}</div>`
}
```

This works exactly like `data()` - `msg` becomes a reactive and mutable property, but **only on the render context.** In order to expose a reactive value that can be mutated by a function declared inside `setup()`, we can use the `value` API:

``` js
import { value } from 'vue'

const MyComponent = {
  setup(props) {
    const msg = value('hello')
    const appendName = () => {
      msg.value = `hello ${props.name}`
    }
    return {
      msg,
      appendName
    }
  },
  template: `<div @click="appendName">{{ msg }}</div>`
}
```

Calling `value()` returns a **value wrapper** object that contains a single reactive property: `.value`. This property points to the actual value the wrapper is holding - in the example above, a string. The value can be mutated:

 ``` js
// read the value
console.log(msg.value) // 'hello'
 // mutate the value
msg.value = 'bye'
```

### Why do we need value wrappers?

Primitive values in JavaScript like numbers and strings are not passed by reference. Returning a primitive value from a function means the receiving function will not be able to read the latest value when the original is mutated or replaced.

Value wrappers are important because they provide a way to pass around mutable and reactive references for arbitrary value types. This is what enables composition functions to encapsulate the logic that manages the state while passing the state back to the components as a trackable reference:

``` js
setup() {
  const valueA = useLogicA() // logic inside useLogicA may mutate valueA
  const valueB = useLogicB()
  return {
    valueA,
    valueB
  }
}
```

Value wrappers can also hold non-primitive values and will make all nested properties reactive. Holding non-primitive values like objects and arrays inside a value wrapper provides the ability to entirely replace the value with a fresh one:

``` js
const numbers = value([1, 2, 3])
// replace the array with a filtered copy
numbers.value = numbers.value.filter(n => n > 1)
```

If you want to create a non-wrapped reactive object, use `state` (which is an exact equivalent of 2.x `Vue.observable` API):

``` js
import { state } from 'vue'

const object = state({
  count: 0
})

object.count++
```

### Value Unwrapping

Note in the last example we are using `{{ msg }}` in the template without the `.value` property access. This is because **value wrappers get "unwrapped" when they are accessed on the render context or as a nested property inside a reactive object.**

You can mutate an unwrapped value binding in inline handlers:

``` js
const MyComponent = {
  setup() {
    return {
      count: value(0)
    }
  },
  template: `<button @click="count++">{{ count }}</button>`
}
```

Value wrappers are also automatically unwrapped when accessed as a nested property inside a reactive object:

``` js
const count = value(0)
const obj = observable({
  count
})

console.log(obj.count) // 0

obj.count++
console.log(obj.count) // 1
console.log(count.value) // 1

count.value++
console.log(obj.count) // 2
console.log(count.value) // 2
```

As a rule of thumb, the only occasions where you need to use `.value` is when directly accessing value wrappers as variables.

### Computed Values

In addition to plain value wrappers, we can also create computed values:

 ``` js
import { value, computed } from 'vue'

const count = value(0)
const countPlusOne = computed(() => count.value + 1)

console.log(countPlusOne.value) // 1

count.value++
console.log(countPlusOne.value) // 2
```

A computed value behaves just like a 2.x computed property: it tracks its dependencies and only re-evaluates when dependencies have changed.

Computed values can also be returned from `setup()` and will get unwrapped just like normal value wrappers. The main difference is that they are read-only by default - assigning to a computed value's `.value` property or attempting to mutate a computed value binding on the render context will be a no-op and result in a warning.

To create a writable computed value, provide a setter via the second argument:

``` js
const count = value(0)
const writableComputed = computed(
  // read
  () => count.value + 1,
  // write
  val => {
    count.value = val - 1
  }
)
```

## Watchers

The `watch` API provides a way to perform side effect based on reactive state changes.

The first argument passed to `watch` is called a "source", which can be either a getter function, a value wrapper, or an array containing either. The second argument is a callback that will only get called when the value returned from the getter or the value wrapper has changed:

``` js
watch(
  // getter
  () => count.value + 1,
  // callback
  (value, oldValue) => {
    console.log('count + 1 is: ', value)
  }
)
// -> count + 1 is: 1

count.value++
// -> count + 1 is: 2
```

Unlike 2.x `$watch`, the callback will be called once when the watcher is first created. This is similar to 2.x watchers with `immediate: true`, but with a slight difference. **By default, the callback is called after current renderer flush.** In other words, the callback is always called when the DOM has already been updated. [This behavior can be configured](#watcher-callback-timing).

In 2.x we often notice code that performs the same logic in `mounted` and in a watcher callback - e.g. fetching data based on a prop. The new `watch` behavior makes it achievable with a single statement.

### Watching Props

As mentioned previously, the `props` object passed to the `setup()` function is reactive and can be used to watch for props changes:

``` js
const MyComponent = {
  props: {
    id: number
  },
  setup(props) {
    const data = value(null)
    watch(() => props.id, async (id) => {
      data.value = await fetchData(id)
    })
  }
}
```

### Watching Value Wrappers

As mentioned, `watch` can watch a value wrapper directly.

``` js
// double is a computed value
const double = computed(() => count.value * 2)

// watch a value directly
watch(double, value => {
  console.log('double the count is: ', value)
}) // -> double the count is: 0

count.value++ // -> double the count is: 2
```

### Watching Multiple Sources

`watch` can also watch an array of sources. Each source can be either a getter function or a value wrapper. The callback receives an array containing the resolved value for each source:

``` js
watch(
  [valueA, () => valueB.value],
  ([a, b], [prevA, prevB]) => {
    console.log(`a is: ${a}`)
    console.log(`b is: ${b}`)
  }
)
```

### Stopping a Watcher

A `watch` call returns a stop handle:

``` js
const stop = watch(...)
// stop watching
stop()
```

If `watch` is called inside `setup()` or lifecycle hooks of a component instance, it will automatically be stopped when the associated component instance is unmounted:

``` js
export default {
  setup() {
    // stopped automatically when the component unmounts
    watch(/* ... */)
  }
}
```

### Effect Cleanup

Sometimes the watcher callback will perform async side effects that need to be invalidated when the watched value changes. The watcher callback receives a 3rd argument that can be used to register a cleanup function. The cleanup function is called when:

- the watcher is about to re-run
- the watcher is stopped (i.e. when the component is unmounted if `watch` is used inside `setup()`)

``` js
watch(idValue, (id, oldId, onCleanup) => {
  const token = performAsyncOperation(id)
  onCleanup(() => {
    // id has changed or watcher is stopped.
    // invalidate previously pending async operation
    token.cancel()
  })
})
```

We are registering cleanup via a passed-in function instead of returning it from the callback (like React `useEffect`) because the return value is important for async error handling. It is very common for the watcher callback to be an async function when performing data fetching:

``` js
const data = value(null)
watch(getId, async (id) => {
  data.value = await fetchData(id)
})
```

An async function implicitly returns a Promise, but the cleanup function needs to be registered immediately before the Promise resolves. In addition, Vue relies on the returned Promise to automatically handle potential errors in the Promise chain.

### Watcher Callback Timing

By default, all watcher callbacks are fired **after current renderer flush.** This ensures that when callbacks are fired, the DOM will be in already-updated state. If you want a watcher callback to fire before flush or synchronously, you can use the `flush` option:

``` js
watch(
  () => count.value + 1,
  () => console.log(`count changed`),
  {
    flush: 'post', // default, fire after renderer flush
    flush: 'pre', // fire right before renderer flush
    flush: 'sync' // fire synchronously
  }
)
```

### Full `watch` Options

``` ts
interface WatchOptions {
  lazy?: boolean
  deep?: boolean
  flush?: 'pre' | 'post' | 'sync'
  onTrack?: (e: DebuggerEvent) => void
  onTrigger?: (e: DebuggerEvent) => void
}

interface DebuggerEvent {
  effect: ReactiveEffect
  target: any
  key: string | symbol | undefined
  type: 'set' | 'add' | 'delete' | 'clear' | 'get' | 'has' | 'iterate'
}
```

- `lazy` is the opposite of 2.x's `immediate` option.
- `deep` works the same as 2.x
- `onTrack` and `onTrigger` are hooks that will be called when a dependency is tracked or the watcher getter is triggered. They receive a debugger event that contains information on the operation that caused the track / trigger.

## Lifecycle Hooks

All current lifecycle hooks will have an equivalent `useXXX` function that can be used inside `setup()`:

``` js
import { onMounted, onUpdated, onUnmounted } from 'vue'

const MyComponent = {
  setup() {
    onMounted(() => {
      console.log('mounted!')
    })
    onUpdated(() => {
      console.log('updated!')
    })
    onUnmounted(() => {
      console.log('unmounted!')
    })
  }
}
```

## Dependency Injection

``` js
import { provide, inject } from 'vue'

const CountSymbol = Symbol()

const Ancestor = {
  setup() {
    // providing a value can make it reactive
    const count = value(0)
    provide({
      [CountSymbol]: count
    })
  }
}

const Descendent = {
  setup() {
    const count = inject(CountSymbol)
    return {
      count
    }
  }
}
```

If provided key contains a value wrapper, `inject` will also return a value wrapper and the binding will be reactive (i.e. the child will update if ancestor mutates the provided value).

# Drawbacks

- Makes it more difficult to reflect and manipulate component definitions.

  This might be a good thing since reflecting and manipulation of component options is usually fragile and risky in a userland context, and creates many edge cases for the runtime to handle (especially when extending or using mixins). The flexibility of function APIs should be able to achieve the same end goals with more explicit userland code.

- Undisciplined users may end up with "spaghetti code" since they are no longer forced to separate component code into option groups.

  // TODO

# Alternatives

- [Class API](https://github.com/vuejs/rfcs/pull/17) (dropped)

# Adoption strategy

The proposed APIs are all new additions and can theoretically be introduced in a completely backwards compatible way. However, the new APIs can replace many of the existing options and makes them unnecessary in the long run. Being able to drop some of these old options will result in considerably smaller bundle size and better performance.

Therefore we are planning to provide two builds for 3.0:

- **Compatibility build**: supports both the new function-based APIs AND all the 2.x options.

- **Standard build**: supports the new function-based APIs and only a subset of 2.x options.

In the compatibility build, `setup()` can be used alongside 2.x options. Note that `setup()` will be called before `data`, `computed` and `method` options are resolved - i.e. you can access values returned from `setup()` on `this` in these options, but not the other way around.

Current 2.x users can start with the compatibility build and progressively migrate away from deprecated options, until eventually switching to the standard build.

### Preserved Options

> Preserved options work the same as 2.x and are available in both the compatibility and standard builds of 3.0. Options marked with * may receive further adjustments before 3.0 official release.

- `name`
- `props`
- `template`
- `render`
- `components`
- `directives`
- `filters` *
- `delimiters` *
- `comments` *

### Options deprecated by this RFC

> These options will only be available in the compatibility build of 3.0.

- `data` (replaced by `setup()`)
- `computed` (replaced by `computed` returned from `setup()`)
- `methods` (replaced by plain functions returned from `setup()`)
- `watch` (replaced by `watch`)
- `provide/inject` (replaced by `provide` and `inject`)
- `mixins` (replaced by function composition)
- `extends` (replaced by function composition)
- All lifecycle hooks (replaced by `onXXX` functions)

### Options deprecated by other RFCs

> These options will only be available in the compatibility build of 3.0.

- `el`

  Components are no longer mounted by instantiating a constructor with `new`, Instead, a root app instance is created and explicitly mounted. See [RFC#29](https://github.com/vuejs/rfcs/blob/global-api-change/active-rfcs/0000-global-api-change.md#mounting-app-instance).

- `propsData`

  Props for root component can be passed via app instance's `mount` method. See [RFC#29](https://github.com/vuejs/rfcs/blob/global-api-change/active-rfcs/0000-global-api-change.md#mounting-app-instance).

- `functional`

  Functional components are now declared as plain functions. See [RFC#27](https://github.com/vuejs/rfcs/pull/27).

- `model`

  No longer necessary with `v-model` arguments. See [RFC#31](https://github.com/vuejs/rfcs/pull/31).

- `inheritAttrs`

  Deperecated by [RFC#26](https://github.com/vuejs/rfcs/pull/26).

# Appendix

## Comparison with React Hooks

The function based API provides the same level of logic composition capabilities as React Hooks, but with some important differences. Unlike React hooks, the `setup()` function is called only once. This means code using Vue's function APIs are:

- In general more aligned with the intuitions of idiomatic JavaScript code;
- Not sensitive to call order and can be conditional;
- Not called repeatedly on each render and produce less GC pressure;
- Not subject to the issue where `useEffect` callback may capture stale variables if the user forgets to pass the correct dependency array;
- Not subject to the issue where `useMemo` is almost always needed in order to prevent inline handlers causing over-re-rendering of child components;

## Type Issues with Class API

The primary goal of introducing the Class API was to provide an alternative API that comes with better TypeScript inference support. However, the fact that Vue components need to merge properties declared from multiple sources onto a single `this` context creates a bit of a challenge even with a Class-based API.

One example is the typing of props. In order to merge props onto `this`, we have to either use a generic argument to the component class, or use a decorator.

Here's an example using generic arguments:

``` ts
interface Props {
  message: string
}

class App extends Component<Props> {
  static props = {
    message: String
  }
}
```

Since the interface passed to the generic argument is in type-land only, the user still needs to provide a runtime props declaration for the props proxying behavior on `this`. This double-declaration is redundant and awkward.

We've considered using decorators as an alternative:

``` ts
class App extends Component<Props> {
  @prop message: string
}
```

Using decorators creates a reliance on a stage-2 spec with a lot of uncertainties, especially when TypeScript's current implementation is completely out of sync with the TC39 proposal. In addition, there is no way to expose the types of props declared with decorators on `this.$props`, which breaks TSX support. Users may also assume they can declare a default value for the prop with `@prop message: string = 'foo'` when technically it just can't be made to work as expected.

In addition, currently there is no way to leverage contextual typing for the arguments of class methods - which means the arguments passed to a Class' `render` function cannot have inferred types based on the Class' other properties.
