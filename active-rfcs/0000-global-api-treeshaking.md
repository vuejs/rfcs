- Start Date: 2019-03-01
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Make Vue runtime tree-shakable by exposing as much APIs through named exports as possible.

# Basic example

``` js
import { nextTick, observable } from 'vue'

nextTick(() => {})

const obj = observable({})
```

# Motivation

As Vue's API grows, we are constantly trying to balance the tradeoff between features and bundle sizes. We want to keep Vue's size overhead to a minimum, but we also don't want to limit its capability because of the size constraint.

With ES modules' static analysis friendly design, modern bundlers combined with minifiers can now eliminate ES modules exports that are not used anywhere in the bundle. We can restructure Vue's global and internal APIs to take advantage of this so that users only pay for the features they actually use.

In addition, knowing that optional features won't increase the bundle size for users not using them, we now have more room to include optional features in the core.

# Detailed design

Currently in 2.x, all global APIs are exposed on the single Vue object:

``` js
import Vue from 'vue'

Vue.nextTick(() => {})

const obj = Vue.observable({})
```

In 3.x, they can **only** be accessed as named imports:

``` js
import Vue, { nextTick, observable } from 'vue'

Vue.nextTick // undefined

nextTick(() => {})

const obj = observable({})
```

By not attaching all APIs on the `Vue` default export, any unused APIs can be dropped in the final bundle produced by a bundler that supports tree-shaking.

In addition to public APIs, many of the internal components / helpers can be exported as named exports as well. This allows the compiler to output code that only imports features when they are used. For example the following template:

``` html
<transition>
  <div v-show="ok">hello</div>
</transition>
```

Can be compiled into the following (for explanation purposes, not exact output):

``` js
import { h, Transition, applyDirectives, vShow } from 'vue'

export function render() {
  return h(Transition, [
    applyDirectives(h('div', 'hello'), this, [vShow, this.ok])
  ])
}
```

This means the `Transition` component only gets imported when the application actually makes use of it.

**Note the above only applies to the ES Modules builds for use with tree-shaking capable bundlers - the UMD build still includes all features and exposes everything on the `Vue` global variable (and the compiler will produce appropriate output to use APIs off the global instead of importing).**

# Drawbacks

Users can no longer import a single `Vue` variable and then use APIs off of it. However this should be a worthwhile tradeoff for minimal bundle sizes.

# Alternatives

N/A

# Adoption strategy

It should be possible to provide a code mod for this as part of the migration tool.
