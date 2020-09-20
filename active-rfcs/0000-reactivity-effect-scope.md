- Start Date: 2020-08-20
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

A new `effectScope` API for `@vue/reactivity` that auto collects the effects created in a function call, and returns a stop handler which clean up all the effects in that scope.

# Basic example

```ts
// effect, computed, watch, watchEffect created inside the scope will be auto collected
const stop = effectScope(() => {
  const doubled = computed(() => counter.value * 2)

  watch(doubled, () => console.log(double.value))

  watchEffect(() => console.log('Count: ', double.value))
})

// to dispose all effects in the scope
stop()
```

# Motivation

When we use Composition API in the component setup, effects will be collected and bound to the current instance, when the instance get unmounted, this effects will be disposed automatically which I think it's very awesome and convenient.

However,  when we are using them out side of components or as a standalone package, it's not that ease. For example, this would be how we might need to do for disposing the effects of `computed` & `watch`

```ts
const disposables = []

const counter = ref(0)
const doubled = computed(() => counter.value * 2)

disposables.push(() => stop(doubled.effect))

cosnt stopWatch1 = watchEffect(() => {
  console.log(`counter: ${counter.value}`
}))

disposables.push(stopWatch1)

cosnt stopWatch2 = watch(doubled, () => console.log(double.value))

disposables.push(stopWatch2)
```

And to stop the effects:

```ts
disposables.forEach(f => f())
disposables = []
```

Especially when we have some long and complex composable code, it's laborious to manually collect all the effects and it's easy to forget some (or you don't have access to effects created in the composable functions) which might resulted unexpected behavior and will be hard to debug.

# Detailed design

Similar to what we did for `currentInstance`, on effect creation, if the `activeEffectScope` is presented, push itself into the scope.

```ts
function createReactiveEffect( /* ... */ ) {
  const effect = function reactiveEffect(): unknown {
    /* ... */ 
  } as ReactiveEffect
  /* ... */
  
  if (activeEffectScope)
    activeEffectScope.effects.push(effect) // <—
  
  return effect
}
```

### Nested Scopes

Nested scopes should also be collectable to be it's parent scope

```ts
const stop = effectScope(() => {
  const doubled = computed(() => counter.value * 2)
  
  // not need to get the stop handler, it will be collected by the outer scope
  effectScope(() => {
    watch(doubled, () => console.log(double.value))
  })

  watchEffect(() => console.log('Count: ', double.value))
})

// to dispose all effects, as well as the nested scopes
stop()
```

### Implementation

I have implement a third-party library [@vue-reactivity/scope](https://github.com/vue-reactivity/scope) which is aimed to solve the exact same roblem. However, [it wrapped & re-exported](https://github.com/vue-reactivity/scope/blob/master/src/hijack.ts#L11) the `effect` and `computed` etc to make the auto collecting work. This deviated from the Vue's design and can not be directly reused for Vue applications. As this is a quite low level feature, I think it's better to be done in `@vue/reactivity` itself.

### Affects to Vue

In `@vue/runtime-dom`, [it wrapped the `computed` to add the instance binding](https://github.com/vuejs/vue-next/blob/master/packages/runtime-core/src/apiComputed.ts). This make the following statements NOT equivalent

```ts
import { computed } from '@vue/reactivity'
import { computed } from 'vue'
```  

This should not be an issue for most of the users, but for some libraries that would like to only rely on `@vue/reactivity` (for more flexible usages) this my cause some minor difference.

With this RFC, `@vue/runtime-dom` might also be able to do some refactoring by using the `effectScope` to collect those effects transparently. So the computed wrapping might not be necessary anymore.

# Drawbacks

- doesn't work well with async scope function
- variables created in the closure will be a bit hard to be accessible for the outer closure. (or maybe we should accept it's return value and passed out?)

# Alternatives

Maybe a `onEffectCreation` hook, which can provide more flexibility and let users easily create their own scope utils or their custom frameworks' lifecycles. But that might be too flexible.

# Adoption strategy

This is a new API and should not affect to existing code.

# Unresolved questions

- Naming
- Migration into `@vue/runtime-dom`
