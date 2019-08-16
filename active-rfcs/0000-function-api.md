- Start Date: 2019-05-30
- Target Major Version: 2.x / 3.x
- Reference Issues:
- Implementation PR: (leave this empty)

# High-level Q&A

## Is this like Python 3 / Angular 2 / Do I have to rewrite all my code?

No. The new API is 100% compatible with current syntax and purely additive. All new additions are contained within the new `setup()` function. **Nothing is being removed or deprecated (in this RFC)**, nor do we have plan to remove / deprecate anything in the foreseeable future. **(A previous draft of this RFC indicated that there is the possibility of deprecating a number of 2.x options in a future major release, which has been redacted based on user feedback.)**

[Details](#adoption-strategy)

## Is this set in stone?

No. This is an RFC (Request for Comments) - as long as this pull request is still open, this is just a proposal for soliciting feedback. We encourage you to voice your opinion, but **please actually read the RFC itself before commenting, as the information you got from a random Reddit/HN thread can be incomplete, outdated or outright misleading.**

## Vue is all about simplicity and this RFC is not.

RFCs are written for implementors and advanced users who are aware of the internal design constraints of the framework. It focuses on the technical details, and has to be extremely thorough and cover all possible edge cases, which is why it may seem complex at first glance.

We will provide tutorials targeting normal users which will be much easier to follow along with. In the meanwhile, check out [some examples](#comparison-with-2x-api) to see if the new API really makes things more complex.

## I don't see what problems this proposal solves.

Please read [this reply](https://github.com/vuejs/rfcs/issues/55#issuecomment-504875870).

## This will lead to spaghetti code and is much harder to read.

Please read [this section](#spaghetti-code-in-unexperienced-hands) and [this reply](https://github.com/vuejs/rfcs/issues/55#issuecomment-504875870).

## The Class API is much better!

We [respectfully](https://github.com/vuejs/rfcs/blob/function-apis/active-rfcs/0000-function-api.md#type-issues-with-class-api) [disagree](https://github.com/vuejs/rfcs/pull/17#issuecomment-494242121).

This RFC also provides strictly superior logic composition and better type inference than the Class API. As it stands, the only "advantage" the Class API has is familiarity - and we don't believe it's enough to outweigh the benefits this RFC provides over it.

## This looks like React, why don't I just use React?

First, the template syntax doesn't change, and you are not forced to use this API for your `<script>` sections at all.

Second, if you use React, you'll most likely be using React Hooks. This API is certainly inspired by React hooks, but it works fundamentally differently and is rooted in Vue's very own reactivity system. In addition, [we believe this API addresses a number of important usability issues in React Hooks](#comparison-with-react-hooks). If you cannot put up with this API, you will most likely dislike React Hooks even more.

# Summary

Expose logic-related component options via function-based APIs instead.

# Basic example

``` vue
<template>
  <div>
    <span>count is {{ count }}</span>
    <span>plusOne is {{ plusOne }}</span>
    <button @click="increment">count++</button>
  </div>
</template>

<script>
import { value, computed, watch, onMounted } from 'vue'

export default {
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
</script>
```

# Motivation

## Logic Composition

One of the key aspects of the component API is how to encapsulate and reuse logic across multiple components. With Vue 2.x's current API, there are a number of common patterns we've seen in the past, each with its own drawbacks. These include:

- Mixins (via the `mixins` option)
- Higher-order components (HOCs)
- Renderless components (via scoped slots)

There is plenty of information regarding these patterns on the internet, so we shall not repeat them in full details here. In general, these patterns all suffer from one or more of the drawbacks below:

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

The function-based APIs, on the other hand, are naturally type-friendly. In the prototype we have already achieved full typing support for the proposed APIs. The best part is - code written in TypeScript will look almost identical to code written in plain JavaScript. More details will be discussed later in this RFC.

See also:

- [Appendix: Type Issues with Class API](#type-issues-with-class-api)

## Bundle Size

Function-based APIs are exposed as named ES exports and imported on demand. This makes them tree-shakable, and leaves more room for future API additions. Code written with function-based APIs also compresses better than object-or-class-based code, since (with standard minification) function and variable names can be shortened while object/class methods and properties cannot.

# Detailed design

## The `setup` function

A new component option, `setup()` is introduced. As the name suggests, this is the place where we use the function-based APIs to setup the logic of our component. `setup()` is called when an instance of the component is created, after props resolution. The function receives the resolved props as its first argument:

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

The second argument provides a context object which exposes a number of properties that were previously exposed on `this` in 2.x APIs:

``` js
const MyComponent = {
  setup(props, context) {
    context.attrs
    context.slots
    context.refs
    context.emit
    context.parent
    context.root
  }
}
```

`attrs`, `slots` and `refs` are in fact proxies to the corresponding values on the internal component instance. This ensures they always expose the latest values even after updates, so we can destructure them without worrying accessing a stale reference:

``` js
const MyComponent = {
  setup(props, { refs }) {
    // a function that may get called at a later stage
    function onClick() {
      refs.foo // guaranteed to be the latest reference
    }
  }
}
```

Why don't we expose `props` via context as well, so that `setup()` needs just a single argument? There are several reasons for this:

- It's much more common for a component to use `props` than the other properties, and very often a component uses only `props`.

- Having `props` as a separate argument makes it easier to type it individually (see [TypeScript-only Props Typing](#typescript-only-props-typing) below) without messing up the types of other properties on the context. It also makes it possible to keep a consistent signature across `setup`, `render` and plain functional components with TSX support.

**`this` is not available inside `setup()`.** The reason for avoiding `this` is because of a very common pitfall for beginners:

``` js
setup() {
  function onClick() {
    this // not the `this` you'd expect!
  }
}
```

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

Note in the last example we are using `{{ msg }}` in the template without the `.value` property access. This is because **value wrappers get "unwrapped" when they are accessed in the template or as a nested property inside a reactive object.**

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
const obj = state({
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

### Usage with Manual Render Functions

If the component doesn't use a template, `setup()` can also directly return a render function instead:

``` js
import { value, createElement as h } from 'vue'

const MyComponent = {
  setup(initialProps) {
    const count = value(0)
    const increment = () => { count.value++ }

    return (props, slots, attrs, vnode) => (
      h('button', {
        onClick: increment
      }, count.value)
    )
  }
}
```

The returned render function has the same signature as specified in [RFC#28](https://github.com/vuejs/rfcs/pull/28).

You may notice that both `setup()` and the returned function receive props as the first argument. They work mostly the same, but the `props` passed to the render function is a plain object in production and offers better performance.

A normal `render()` option is still available, but is mostly used as a result of template compilation. For manual render functions, an inline function returned from `setup()` should be preferred since it avoids the need for proxying bindings and makes type inference easier.

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

The first argument passed to `watch` is called a "source", which can be one of the following:

- a getter function
- a value wrapper
- an array containing the two above types

The second argument is a callback that will only get called when the value returned from the getter or the value wrapper has changed:

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
    id: Number
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

All current lifecycle hooks will have an equivalent `onXXX` function that can be used inside `setup()`:

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

`provide` and `inject` functions do the same job `provide` and `inject` options did in Vue 2.x.
The `provide` function takes object as a first argument or key (`string`, `Symbol` or `Key<T>`) as first argument and value as the second one.
The `inject` function takes keys as a rest array `(...keys)` and returns an object containing values for all of the keys.

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
    const { count } = inject(CountSymbol)
    return {
      count
    }
  }
}
```

If provided key contains a value wrapper, `inject` will also return a value wrapper and the binding will be reactive (i.e. the child will update if ancestor mutates the provided value).

## Type Inference

To get proper type inference in TypeScript, we do need to wrap a component definition in a function call:

``` ts
import { createComponent } from 'vue'

const MyComponent = createComponent({
  // props declarations are used to infer prop types
  props: {
    msg: String
  },
  setup(props) {
    props.msg // string | undefined

    // bindings returned from setup() can be used for type inference
    // in templates
    const count = value(0)
    return {
      count
    }
  }
})
```

`createComponent` is conceptually similar to 2.x's `Vue.extend`, but it is a no-op and only needed for typing purposes. The returned component is the object itself, but typed in a way that would provide type information for Vetur and TSX. If you are using Single-File Components, Vetur can implicitly add the wrapper function for you.

If you are using render functions / TSX, returning a render function inside `setup()` provides proper type support (again, no manual type hints needed):

``` ts
import { createComponent, createElement as h } from 'vue'

const MyComponent = createComponent({
  props: {
    msg: String
  },
  setup(props) {
    const count = value(0)
    return () => h('div', [
      h('p', `msg is ${props.msg}`),
      h('p', `count is ${count.value}`)
    ])
  }
})
```

### TypeScript-only Props Typing

In 3.0, the `props` declaration is optional. If you don't want runtime props validation, you can omit `props` declaration and declare your expected prop types directly in TypeScript:

``` ts
import { createComponent, createElement as h } from 'vue'

interface Props {
  msg: string
}

const MyComponent = createComponent({
  setup(props: Props) {
    return () => h('div', props.msg)
  }
})
```

You can even pass the setup function directly if you don't need any other options:

``` ts
const MyComponent = createComponent((props: { msg: string }) => {
  return () => h('div', props.msg)
})
```

The returned `MyComponent` also provides type inference when used in TSX.

### Required Props

By default, props are inferred as optional properties. `required: true` will be respected if present:

``` ts
import { createComponent } from 'vue'

createComponent({
  props: {
    foo: {
      type: String,
      required: true
    },
    bar: {
      type: String
    }
  } as const,
  setup(props) {
    props.foo // string
    props.bar // string | undefined
  }
})
```

Note that we need to add `as const` after the `props` declaration. This is because without `as const` the type will be `required: boolean` and won't qualify for `extends true` in conditional type operations.

> Side note: should we consider making props required by default (And can be made optional with `optional: true`)?

### Complex Prop Types

The exposed `PropType` type can be used to declare complex prop types - but it requires a force-cast via `as any`:

``` ts
import { createComponent, PropType } from 'vue'

createComponent({
  props: {
    options: (null as any) as PropType<{ msg: string }>
  },
  setup(props) {
    props.options // { msg: string } | undefined
  }
})
```

### Dependency Injection Typing

`provide` and `inject` can be typed by providing a typed symbol using the `Key` type:

``` ts
import { createComponent, provide, inject, Key } from 'vue'

const CountSymbol: Key<number> = Symbol()

const Provider = createComponent({
  setup() {
    // will error if provided value is not a number
    provide(CountSymbol, 123)
  }
})

const Consumer = createComponent({
  setup() {
    const count = inject(CountSymbol) // count's type is Value<number>
    console.log(count.value) // 123
    return {
      count
    }
  }
})
```

# Drawbacks

### Runtime Reflection of Components

The new API makes it more difficult to reflect and manipulate component definitions. This might be a good thing since reflecting and manipulation of component options is usually fragile and risky in a userland context, and creates many edge cases for the runtime to handle (especially when extending or using mixins). The flexibility of function APIs should be able to achieve the same end goals with more explicit userland code.

### Spaghetti Code in Unexperienced Hands

Some feedbacks suggest that undisciplined users may end up with "spaghetti code" since they are no longer forced to separate component code into option groups. I believe this fear is unwarranted. It is true that the flexibility of function-based API will theoretically allow users to write code that is harder to follow. But let me explain why this is unlikely to happen.

  The biggest difference of function-based APIs vs. the current option-based API is that function APIs make it ridiculously easy to extract part of your component logic into a well encapsulated function. This can be done not just for reuse, but purely for code organization purposes as well.

  With component options, your code only *seem* to be organized - in a complex component, logic related to a specific task is often split up between multiple options. For example, fetching a piece of data often involves one or more properties in `props` and `data()`, a `mounted` hook, and a watcher declared in `watch`. If you put all the logic of your app in a single component, that component is going to become a monster and become very hard to maintain, because every single logical task will be fragmented and spanning across multiple option blocks.

  In comparison, with function-based API all the code related to fetching a specific piece of data can be nicely encapsulated in a single function.

  To avoid the "monster component" problem, we split the component into many smaller ones. Similarly, if you have a huge `setup()` function, you can split it into multiple functions, each dealing with a specific logical task. Function based API makes better organized code easily possible while with options you are stuck with... options (because splitting into mixins makes things worse).

  From this perspective, separation of options vs. `setup()` is just like the separation of HTML/CSS/JS vs. Single File Components.

# Alternatives

- [Class API](https://github.com/vuejs/rfcs/pull/17) (dropped)

# Adoption strategy

The proposed API is purely additive and can be introduced in a completely backwards compatible fashion. No existing code needs to be rewritten (this excludes breaking changes introduced in other RFCs).

We plan to first introduce the proposed API as a standalone plugin for 2.x (before merging this RFC) so that users can experiment with the API and provide more concrete feedback. **Release of the plugin is not an indication that the API will eventually become part of Vue** - it's only for collecting feedback for this RFC.

Assuming this RFC eventually lands, in 3.0 `setup()` can be used alongside 2.x options. Note that `setup()` will be called before `data`, `computed` and `method` options are resolved - i.e. you can access values returned from `setup()` on `this` in these options, but not the other way around.

# Appendix

## Comparison with 2.x API

### Simple Counter

Standard API

``` vue
<template>
  <div>
    Count is {{ count }}, count * 2 is {{ double }}
    <button @click="increment">+</button>
  </div>
</template>

<script>
export default {
  data() {
    return {
      count: 0
    }
  },
  methods: {
    increment() {
      this.count++
    }
  },
  computed: {
    double() {
      return this.count * 2
    }
  }
}
</script>
```

Functions API

``` vue
<template>
  <div>
    Count is {{ count }}, count * 2 is {{ double }}
    <button @click="increment">+</button>
  </div>
</template>

<script>
import { value, computed } from 'vue'

export default {
  setup() {
    const count = value(0)
    const double = computed(() => count.value * 2)
    const increment = () => { count.value++ }
    return {
      count,
      double,
      increment
    }
  }
}
</script>
```

### Fetching Data Based on Prop

Standard API

``` vue
<template>
  <div>
    <template v-if="isLoading">Loading...</template>
    <template v-else>
      <h3>{{ post.title }}</h3>
      <p>{{ post.body }}</p>
    </template>
  </div>
</template>

<script>
import { fetchPost } from './api'

export default {
  props: {
    id: Number
  },
  data() {
    return {
      isLoading: true,
      post: null
    }
  },
  mounted() {
    this.fetchPost()
  },
  watch: {
    id: 'fetchPost'
  },
  methods: {
    async fetchPost() {
      this.isLoading = true
      this.post = await fetchPost(this.id)
      this.isLoading = false
    }
  }
}
</script>
```

Functions API

``` vue
<template>
  <div>
    <template v-if="isLoading">Loading...</template>
    <template v-else>
      <h3>{{ post.title }}</h3>
      <p>{{ post.body }}</p>
    </template>
  </div>
</template>

<script>
import { value, watch } from 'vue'
import { fetchPost } from './api'

export default {
  setup(props) {
    const isLoading = value(true)
    const post = value(null)

    watch(() => props.id, async (id) => {
      isLoading.value = true
      post.value = await fetchPost(id)
      isLoading.value = false
    })

    return {
      isLoading,
      post
    }
  }
}
</script>
```

### Multiple Logic Topics

Based on the previous data-fetching example, suppose we want to also track mouse position in the same component:

Standard API

``` vue
<template>
  <div>
    <template v-if="isLoading">Loading...</template>
    <template v-else>
      <h3>{{ post.title }}</h3>
      <p>{{ post.body }}</p>
    </template>
    <div>Mouse is at {{ x }}, {{ y }}</div>
  </div>
</template>

<script>
import { fetchPost } from './api'

export default {
  props: {
    id: Number
  },
  data() {
    return {
      isLoading: true,
      post: null,
      x: 0,
      y: 0
    }
  },
  mounted() {
    this.fetchPost()
    window.addEventListener('mousemove', this.updateMouse)
  },
  watch: {
    id: 'fetchPost'
  },
  destroyed() {
    window.removeEventListener('mousemove', this.updateMouse)
  },
  methods: {
    async fetchPost() {
      this.isLoading = true
      this.post = await fetchPost(this.id)
      this.isLoading = false
    },
    updateMouse(e) {
      this.x = e.pageX
      this.y = e.pageY
    }
  }
}
</script>
```

You'll start to notice that we have two logic topics (data fetching and mouse position tracking) but they are split up and mixed between component options.

With the function-based API:

``` vue
<template>
  <div>
    <template v-if="isLoading">Loading...</template>
    <template v-else>
      <h3>{{ post.title }}</h3>
      <p>{{ post.body }}</p>
    </template>
    <div>Mouse is at {{ x }}, {{ y }}</div>
  </div>
</template>

<script>
import { value, watch, onMounted, onUnmounted } from 'vue'
import { fetchPost } from './api'

function useFetch(props) {
  const isLoading = value(true)
  const post = value(null)

  watch(() => props.id, async (id) => {
    isLoading.value = true
    post.value = await fetchPost(id)
    isLoading.value = false
  })

  return {
    isLoading,
    post
  }
}

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

export default {
  setup(props) {
    return {
      ...useFetch(props),
      ...useMouse()
    }
  }
}
</script>
```

Notice how the function-based API cleanly organizes code by logical topic instead of options.

More examples can be found in [this gist](https://gist.github.com/yyx990803/762ec427882a61be3e4affe02f8af555).

#### Hybrid Approaches work too

It is also possible to encapsulate one logic topic in `setup` while otherwise using the Standard API. In this example, mouse-related logic is handled with the Function-based API while the fetching logic is handled with the Standard API.

_When `setup` is used alongside the Standard API, `setup` runs first and any conflicting properties are overridden by the Standard API._

```vue
<template>
  <div>
    <template v-if="isLoading">Loading...</template>
    <template v-else>
      <h3>{{ post.title }}</h3>
      <p>{{ post.body }}</p>
    </template>
    <div>Mouse is at {{ x }}, {{ y }}</div>
  </div>
</template>

<script>
import { value, onMounted, onUnmounted } from 'vue'
import { fetchPost } from './api'

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

export default {
  props: {
    id: Number
  },
  setup(props) {
    return {
      ...useMouse()
    }
  },
  data() {
    return {
      isLoading: true,
      post: null,
    }
  },
  mounted() {
    this.fetchPost()
  },
  watch: {
    id: 'fetchPost'
  },
  methods: {
    async fetchPost() {
      this.isLoading = true
      this.post = await fetchPost(this.id)
      this.isLoading = false
    }
  }
}
</script>
```

## Comparison with React Hooks

The function based API provides the same level of logic composition capabilities as React Hooks, but with some important differences. Unlike React hooks, the `setup()` function is called only once. This means code using Vue's function APIs are:

- In general more aligned with the intuitions of idiomatic JavaScript code;
- Not sensitive to call order and can be conditional;
- Not called repeatedly on each render and produce less GC pressure;
- Not subject to the issue where `useCallback` is almost always needed in order to prevent inline handlers causing over-re-rendering of child components;
- Not subject to the issue where `useEffect` and `useMemo` may capture stale variables if the user forgets to pass the correct dependency array. Vue's automated dependency tracking ensures watchers and computed values are always correctly invalidated.

> Note: we acknowledge the creativity of React Hooks, and it is a major source of inspiration for this proposal. However, the issues mentioned above do exist in its design and we noticed Vue's reactivity model happens to provide a way around them.

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
