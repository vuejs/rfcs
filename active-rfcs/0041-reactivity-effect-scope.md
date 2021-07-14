- Start Date: 2020-08-20
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: [#2195](https://github.com/vuejs/vue-next/pull/2195)

# Summary

Introducing a new `effectScope()` API for `@vue/reactivity`. An `EffectScope` instance can automatically collect effects run within a synchronous function so that these effects can be disposed together at a later time.

# Basic example

```ts
// effect, computed, watch, watchEffect created inside the scope will be collected

const scope = effectScope()

scope.run(() => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(double.value))

  watchEffect(() => console.log('Count: ', double.value))
})

// to dispose all effects in the scope
scope.stop()
```

# Motivation

In Vue's component `setup()`, effects will be collected and bound to the current instance. When the instance get unmounted, effects will be disposed automatically. This is a convenient and intuitive feature.

However, when we are using them outside of components or as a standalone package, it's not that simple. For example, this might be what we need to do for disposing the effects of `computed` & `watch`

```ts
const disposables = []

const counter = ref(0)
const doubled = computed(() => counter.value * 2)

disposables.push(() => stop(doubled.effect))

const stopWatch1 = watchEffect(() => {
  console.log(`counter: ${counter.value}`)
})

disposables.push(stopWatch1)

const stopWatch2 = watch(doubled, () => {
  console.log(double.value)
})

disposables.push(stopWatch2)
```

And to stop the effects:

```ts
disposables.forEach((f) => f())
disposables = []
```

Especially when we have some long and complex composable code, it's laborious to manually collect all the effects. It's also easy to forget collecting them (or you don't have access to effects created in the composable functions) which might result in memory leakage and unexpected behavior.

This RFC is trying to abstract the component's `setup()` effect collecting and disposing feature into a more general API that can be reused outside of the component model.

It also provides the functionality to create "detached" effects from the component's `setup()` scope or user-defined scope. Resolving https://github.com/vuejs/vue-next/issues/1532.

# Detailed Design

### New API Summary

- `effectScope(detached = false): EffectScope`

  ```ts
  interface EffectScope {
    run<T>(fn: () => T): T | undefined // undefined if scope is inactive
    stop(): void
  }
  ```

- `getCurrentScope(): EffectScope | undefined`
- `onScopeDispose(fn: () => void): void`

### Basic Usage

Creating a scope:

```ts
const scope = effectScope()
```

A scope can run a function and will capture all effects created during the function's synchronous execution, including any API that creates effects internally, e.g. `computed`, `watch` and `watchEffect`:

```ts
scope.run(() => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(double.value))

  watchEffect(() => console.log('Count: ', double.value))
})

// the same scope can run multiple times
scope.run(() => {
  watch(counter, () => {
    /*...*/
  })
})
```

The `run` method also forwards the return value of the executed function:

```js
console.log(scope.run(() => 1)) // 1
```

When `scope.stop()` is called, it will stop all the captured effects and nested scopes recursively.

```ts
scope.stop()
```

### Nested Scopes

Nested scopes should also be collected by their parent scope. And when the parent scope gets disposed, all its descendant scopes will also be stopped.

```ts
const scope = effectScope()

scope.run(() => {
  const doubled = computed(() => counter.value * 2)

  // not need to get the stop handler, it will be collected by the outer scope
  effectScope().run(() => {
    watch(doubled, () => console.log(double.value))
  })

  watchEffect(() => console.log('Count: ', double.value))
})

// dispose all effects, including those in the nested scopes
scope.stop()
```

### Detached Nested Scopes

`effectScope` accepts an argument to be created in "detached" mode. A detached scope will not be collected by its parent scope.

This also makes usages like ["lazy initialization"](https://github.com/vuejs/vue-next/issues/1532) possible.

```ts
let childScope

const parentScope = effectScope()

parentScope.run(() => {
  const doubled = computed(() => counter.value * 2)

  // with the detected flag,
  // the scope will not be collected and disposed by the outer scope
  childScope = effectScope(true /* detached */)
  childScope.run(() => {
    watch(doubled, () => console.log(double.value))
  })

  watchEffect(() => console.log('Count: ', double.value))
})

// disposes all effects, but not `childScope`
parentScope.stop()

// stop the nested scope only when appropriate
nestedScope.stop()
```

### `onScopeDispose`

The global hook `onScopeDispose()` serves a similar functionality to `onUnmounted()`, but works for the current scope instead of the component instance. This could benefit composable functions to clean up their side effects along with its scope. Since `setup()` also creates a scope for the component, it will be equivalent to `onUnmounted()` when there is no explicit effect scope created.

```ts
import { onScopeDispose } from 'vue'

const scope = effectScope()

scope.run(() => {
  onScopeDispose(() => {
    console.log('cleaned!')
  })
})

scope.stop() // logs 'cleaned!'
```

### Getting the Current Scope

A new API `getCurrentScope()` is introduced to get the current scope.

```ts
import { getCurrentScope } from 'vue'

getCurrentScope() // EffectScope | undefined
```

## Use Case Examples

### Example A: Shared Composable

Some composables setup global side effects. For example the following `useMouse()` function:

```ts
function useMouse() {
  const x = ref(0)
  const y = ref(0)

  function handler(e) {
    x.value = e.x
    y.value = e.y
  }

  window.addEventListener('mousemove', handler)

  onUnmounted(() => {
    window.removeEventListener('mousemove', handler)
  })

  return { x, y }
}
```

If `useMouse()` is called in multiple components, each component will attach a `mousemove` listener and create its own copy of `x` and `y` refs. We should be able to make this more efficient by sharing the same set of listeners and refs across multiple components, but we can't because each `onUnmounted` call is coupled to a single component instance.

We can achieve this using detached scope, and `onScopeDispose`. First, we need to replace `onUnmounted` with `onScopeDispose`:

```diff
- onUnmounted(() => {
+ onScopeDispose(() => {
  window.removeEventListener('mousemove', handler)
})
```

This still works because a Vue component now also runs its `setup()` inside a scope, which will be disposed when the component is unmounted.

Then, we can create a utility function that manages parent scope subscriptions:

```ts
function createSharedComposable(composable) {
  let subscribers = 0
  let state, scope

  const dispose = () => {
    if (scope && --subscribers <= 0) {
      scope.stop()
      state = scope = null
    }
  }

  return (...args) => {
    subscribers++
    if (!state) {
      scope = effectScope(true)
      state = scope.run(() => composable(...args))
    }
    onScopeDispose(dispose)
    return state
  }
}
```

Now we can create a shared version of `useMouse`:

```ts
const useSharedMouse = createSharedComposable(useMouse)
```

The new `useSharedMouse` composable will set up the listener only once no matter how many components are using it, and removes the listener when no component is using it anymore. In fact, the `useMouse` function should probably be a shared composable in the first place!

### Example B: Ephemeral Scopes

```ts
export default {
  setup() {
    const enabled = ref(false)
    let mouseState, mouseScope

    const dispose = () => {
      mouseScope && mouseScope.stop()
      mouseState = null
    }

    watch(
      enabled,
      () => {
        if (enabled.value) {
          mouseScope = effectScope()
          mouseState = mouseScope.run(() => useMouse())
        } else {
          dispose()
        }
      },
      { immediate: true }
    )

    onScopeDispose(dispose)
  },
}
```

In the example above, we would create and dispose some scopes on the fly, `onScopeDispose` allow `useMouse` to do the cleanup correctly while `onUnmounted` would never be called during this process.

## Affected Usage in Vue Core

Currently in `@vue/runtime-dom`, [we wrap the `computed` to add the instance binding](https://github.com/vuejs/vue-next/blob/master/packages/runtime-core/src/apiComputed.ts). This makes the following statements NOT equivalent

```ts
// not the same
import { computed } from '@vue/reactivity'
import { computed } from 'vue'
```

This should not be an issue for most of the users, but for some libraries that would like to only rely on `@vue/reactivity` (for more flexible usages), this might be a pitfall and cause some unwanted side-effects.

With this RFC, `@vue/runtime-dom` can use the `effectScope` to collect the effects directly and computed rewrapping will not be necessary anymore.

```ts
// with the RFC, `vue` simply redirect `computed` from `@vue/reactivity`
import { computed } from '@vue/reactivity'
import { computed } from 'vue'
```

# Drawbacks

- It doesn't work well with async functions

# Alternatives

M/A

# Adoption strategy

This is a new API and should not affect the existing code. It's also a low-level API only intended for advanced users and library authors.

# Unresolved questions

None
