- Start Date: 2021-12-16
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

Introducing a new `addReactivityInterop()` API for `@vue/reactivity`. This feature adds support for (one-way) reactivity interoperability with third party libraries.

# Basic example

```ts
// file: store.js

// use mobx as a demostration of external reactivity source
import { Reaction, makeAutoObservable } from 'mobx'

// register MobX
// this step should usually be performed by _library authors_.
let id = 0
addReactivityInterop((fn, trigger) => {
  const reaction = new Reaction(`externalSource@${++id}`, trigger)
  return {
    track: (x) => {
      let next
      reaction.track(() => (next = fn(x)))
      return next
    },
    dispose: () => reaction.dispose()
  }
})

// user-land mobx store
class Timer {
  secondsPassed = 0

  constructor() {
    makeAutoObservable(this)
  }

  increase() {
    this.secondsPassed += 1
  }

  reset() {
    this.secondsPassed = 0
  }
}

export const timer = new Timer();

setInterval(() => {
    timer.increase();
}, 1000);
```

In vue component:

```html
<script setup>
  import { timer } from './store.js'

  const doubled = computed(() => timer.secondPassed * 2) //even this is possible!
</script>
<template>
  <button @click="timer.reset()">
    Second passed: {{ timer.secondsPassed }}, doubled: {{ doubled }}
  </button>
</template>
```

# Motivation

It adds a lot of convinient, eliminates annoying wrappers (e.g. convert to a `ref`) and we can use third party reactive objects as if they are first-class citizen in vue.

This feature is primarily for __library authors__. Normal users do not need to worry about this.

# Detailed design

## API Summary
```ts
// pending deisgn
type InteropSource<T> = { track: ()=>T , dispose: ()=>void };
type InteropSourceFactory = <T>(fn: ()=>T, trigger: ()=> void) => InteropSource<T>;

function addReactivityInterop(factory: InteropSourceFactory): void;
```

## Implementation

Currently `addReactivityInterop` will perform __irreversible__ side effect: setting the current `InteropSourceFactory` (which should be a global variable).

Whenever a `ReactiveEffect` is constructed, the current `InteropSourceFactory` should be called with the original `fn` passed in `ReactiveEffect.constructor` and a `trigger` function which trigger the `ReactiveEffect` (to be scheduled/re-run). The return value is a `InteropSource`, whose `.track` should replace `.fn` of current `ReactiveEffect` and `.dispose()` should be called whenever current `ReactiveEffect` will be cleaned up. Essentially there are two steps:
* Hook `ReactiveEffect.fn`, so third party libraries get the control to perform their own dependency tracking logic. 
* Cleanup whenever appropriate, to avoid memory leak.


Note this feature implies the (external) dependencies should be transparently collected (by external) and the control should be __NOT__ inversed by external (it's vue taking control at first, calling the third party code to perform their dependency tracking). However, not all reactivity implementations give user the control (usually the control is inversed, e.g. `computed(getter)` whose `getter` is called by framework, not user directly, while mobx's `Reaction.track(fn)` is called by user, and runs immediately) or support auto (transparent/implicit) dependency tracking. 
> It's possible to implement _auto dependency tracking_ and/or _auto subscription_ based on a reactivity implementation which doesn't support these features, yet it's not the topic of this RFC.

### Workaround

In vue 3.2+ there is a work-around by patching `ReactiveEffect`

```ts

let currentInteropSourceFactory:InteropSourceFactory | null; // set by `addReactivityInterop`

Object.defineProperty(ReactiveEffect.prototype, 'fn', {
  get(this: ReactiveEffect & PatchedReactiveEffect) {
    return this._fn
  },
  set(this: ReactiveEffect & PatchedReactiveEffect, originalFn) {
    if(currentInteropSourceFactory) {
      if (this._fn) {
        this._interopSource?.dispose()
      } else {
        this._signal = ref(undefined)
      }
      this._interopSource = currentInteropSourceFactory(originalFn,()=>triggerRef(this._signal));
      this._fn = () => {
        this._signal.value; // track in vue
        return this._interopSource.track();
      } 
    } else {
      this._fn = originalFn;
    }
  }
})

const origianlStop = ReactiveEffect.prototype.stop

Object.defineProperty(ReactiveEffect.prototype, 'stop', {
  value: function (this: ReactiveEffect & PatchedReactiveEffect) {
    this._interopSource?.dispose();
    this._interopSource = null;
    origianlStop.call(this);
  }
});

interface PatchedReactiveEffect {
  _interopSource?: InteropSource;
  _fn?: Function;
  _signal?: Ref<any>;
}
```

## Composed `InteropSourceFactory`

If `addReactivityInterop` is called multiple times, the current `InteropSourceFactory` should be composed.

# Drawbacks

- A bit of magic (but acceptable in the context of vue)
- Add a bit of performance overhead in creation of `ReactiveEffect`

# Alternatives

N/A

# Adoption strategy

This is a new API and should not affect the existing code. It's also a low-level API only intended for advanced users and library authors.

# Unresolved questions

- Better API naming?
