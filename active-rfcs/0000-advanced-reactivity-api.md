- Start Date: 03-05-2019
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Provide standalone APIs for creating and observing reactive state.

# Basic example

```js
import { state, value, computed, watch } from '@vue/observer'

// reactive object
// equivalent of 2.x Vue.observable()
const obj = state({ a: 1 })

// watch with a getter function
watch(() => obj.a, value => {
  console.log(`obj.a is: ${value}`)
})

// a "ref" object that has a .value property
const count = value(0)

// computed "ref" with a read-only .value property
const plusOne = computed(() => count.value + 1)

// refs can be watched directly
watch(count, (count, oldCount) => {
  console.log(`count is: ${count}`)
})

watch(plusOne, countPlusOne => {
  console.log(`count plus one is: ${countPlusOne}`)
})
```

# Motivation

## Decouple the reactivity system from component instances

Vue's reactivity system powers a few aspects of Vue:

- Tracking dependencies used during a component's render for automatic component re-render

- Tracking dependencies of computed properties to only re-compute values when necessary

- Expose `this.$watch` API for users to perform custom side effects in response to state changes

Until 2.6, the reactivity system has largely been considered an internal implementation, and there is no dedicated API for creating / watching reactive state without doing it inside a component instance.

However, such coupling isn't technically necessary. In 3.x we've already split the reactivity system into its own package (`@vue/observer`) with dedicated APIs, so it makes sense to also expose these APIs to enable more advanced use cases.

With these APIs it becomes possible to encapsulate stateful logic and side effects without components involved. In addition, with proper ability to "connect" the created state back into component instances, they also unlock a powerful component logic reuse mechanism.

# Detailed design

## Reactive Objects

In 2.6 we introduced the `observable` API for creating reactive objects. We've noticed the naming causes confusion for some users who are familiar with RxJS or reactive programming where the term "observable" is commonly used to denote event streams. So here we intend to rename it to simply `state`:

``` js
import { state } from 'vue'

const object = state({
  count: 0
})
```

This works exactly like 2.6 `Vue.observable`. The returned object behaves just like a normal object, and when its properties are accessed in **reactive computations** (render functions, computed property getters and watcher getters), they are tracked as dependencies. Mutation to these properties will cause corresponding computations to re-run.

## Value Refs

The `state` API cannot be used for primitive values because:

- Vue tracks dependencies by intercepting property accesses. Usage of primitive values in reactive computations cannot be tracked.

- JavaScript values are not passed by reference. Passing a value directly means the receiving function will not be able to read the latest value when the original is mutated.

The simple solution is wrapping the value in an object wrapper that can be passed around by reference. This is exactly what the `value` API does:

``` js
import { value } from 'vue'

const countRef = value(0)
```

The `value` API creates a wrapper object for a value, called a **ref**. A ref is a reactive object with a single property: `.value`. The property points to the actual value being held and is writable:

``` js
// read the value
console.log(countRef.value) // 0

// mutate the value
countRef.value++
```

Refs are primarily used for holding primitive values, but it can also hold any other values including deeply nested objects and arrays. Non-primitive values held inside a ref behave like normal reactive objects created via `state`.

## Computed Refs

In addition to plain value refs, we can also create computed refs:

``` js
import { value, computed } from 'vue'

const count = value(0)
const countPlusOne = computed(() => count.value + 1)

console.log(countPlusOne.value) // 1
count.value++
console.log(countPlusOne.value) // 2
```

Computed refs are readonly by default - assigning to its `value` property will result in an error.

Computed refs can be made writable by passing a write callback as the 2nd argument:

``` js
const writableRef = computed(
  // read
  () => count.value + 1,
  // write
  val => {
    count.value = val - 1
  }
)
```

Computed refs behaves like computed properties in a component: it tracks its dependencies and only re-evaluates when dependencies have changed.

## Watchers

All `.value` access are reactive, and can be tracked with the standalone `watch` API.

**NOTE: unlike 2.x, the `watch` API is immediate by default.**

`watch` can be called with a single function. The function will be called immediately, and will be called again whenever dependencies change:

``` js
import { value, watch } from 'vue'

const count = value(0)

// watch and re-run the effect
watch(() => {
  console.log('count is: ', count.value)
})
// -> count is: 0

count.value++
// -> count is: 1
```

### Watch with a Getter

When using a single function, any reactive properties accessed during its execution are tracked as dependencies. The **computation** and the **side effect** are performed together. To separate the two, we can pass two functions instead:

``` js
watch(
  // 1st argument (the "computation", or getter) should return a value
  () => count.value + 1,
  // 2nd argument (the "effect", or callback) only fires when value returned
  // from the getter changes
  value => {
    console.log('count + 1 is: ', value)
  }
)
// -> count + 1 is: 1

count.value++
// -> count + 1 is: 2
```

### Watching Refs

The 1st argument can also be a ref:

``` js
// double is a computed ref
const double = computed(() => count.value * 2)

// watch a ref directly
watch(double, value => {
  console.log('double the count is: ', value)
})
// -> double the count is: 0

count.value++
// -> double the count is: 2
```

### Stopping a Watcher

A `watch` call returns a stop handle:

``` js
const stop = watch(...)

// stop watching
stop()
```

If `watch` is called inside lifecycle hooks or `data()` of a component instance, it will automatically be stopped when the associated component instance is unmounted:

``` js
export default {
  created() {
    // stopped automatically when the component unmounts
    watch(() => this.id, id => {
      // ...
    })
  }
}
```

### Effect Cleanup

The **effect** callback can also return a cleanup function which gets called every time when:

- the watcher is about to re-run
- the watcher is stopped

``` js
watch(idRef, id => {
  const token = performAsyncOperation(id)

  return () => {
    // id has changed or watcher is stopped.
    // invalidate previously pending async operation
    token.cancel()
  }
})
```

### Non-Immediate Watchers

To make watchers non-immediate like 2.x, pass additional options via the 3rd argument:

``` js
watch(
  () => count.value + 1,
  () => {
    console.log(`count changed`)
  },
  { immediate: false }
)
```

## Exposing Refs to Components

While this proposal is focused on working with reactive state outside of components, such state should also be usable inside components as well.

Refs can be returned in a component's `data()` function:

``` js
import { value } from 'vue'

export default {
  data() {
    return {
      count: value(0)
    }
  }
}
```

**When a `ref` is returned as a root-level property in `data()`, it is bound to the component instance as a direct property.** This means there's no need to access the value via `.value` - the value can be accessed and mutated directly as `this.count`, and directly as `count` inside templates:

``` html
<div @click="count++">
  {{ count }}
</div>
```

## Beyond the API

The APIs proposed here are just low-level building blocks. Technically, they provide everything we need for global state management, so Vuex can be rewritten as a very thin layer on top of these APIs. In addition, when combined with [the ability to programmatically hook into the component lifecycle](https://github.com/vuejs/rfcs/pull/23), we can offer a logic reuse mechanism with capabilities similar to React hooks.

# Drawbacks

- To pass state around while keeping them "trackable" and "reactive", values must be passed around in the form of ref containers. This is a new concept and can be a bit more difficult to learn than the base API. However, these APIs are intended for advanced use cases so the learning cost should be acceptable.

# Alternatives

N/A

# Adoption strategy

This is mostly new APIs that expose existing internal capabilities. Users familiar with Vue's existing reactivity system should be able to grasp the concept fairly quickly. It should have a dedicated chapter in the official guide, and we also need to revise the [Reactivity in Depth](https://vuejs.org/v2/guide/reactivity.html) section of the current docs.

# Unresolved questions

- `watch` API overlaps with existing `this.$watch` API and `watch` component option. In fact, the standalone `watch` API provides a superset of existing APIs. This makes the existence of all three redundant and inconsistent.

  **Should we deprecate `this.$watch` and `watch` component option?**

  Sidenote: removing `this.$watch` and the `watch` option also makes the entire `watch` API completely tree-shakable.

- We probably need to also expose a `isRef` method to check whether an object is a value/computed ref.
