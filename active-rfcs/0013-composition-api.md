- Start Date: 2019-07-10
- Target Major Version: 2.x / 3.x
- Reference Issues: [#78](https://github.com/vuejs/rfcs/pull/78)
- Implementation PR: N/A

> Since this RFC is long, it is deployed in a more readable format [here](https://vue-composition-api-rfc.netlify.com/). There is also an accompanying [API Reference](https://vue-composition-api-rfc.netlify.com/api.html).

## Summary

Introducing the **Composition API**: a set of additive, function-based APIs that allow flexible composition of component logic.

## Basic example

```html
<template>
  <button @click="increment">
    Count is: {{ state.count }}, double is: {{ state.double }}
  </button>
</template>

<script>
import { reactive, computed } from 'vue'

export default {
  setup() {
    const state = reactive({
      count: 0,
      double: computed(() => state.count * 2)
    })

    function increment() {
      state.count++
    }

    return {
      state,
      increment
    }
  }
}
</script>
```

## Motivation

### Logic Reuse & Code Organization

We all love how Vue is very easy to pick up and makes building small to medium scale applications a breeze. But today as Vue's adoption grows, many users are also using Vue to build large scale projects - ones that are iterated on and maintained over a long timeframe, by a team of multiple developers. Over the years we have witnessed some of these projects run into the limits of the programming model entailed by Vue's current API. The problems can be summarized into two categories:

1. The code of complex components become harder to reason about as features grow over time. This happens particularly when developers are reading code they did not write themselves. The root cause is that Vue's existing API forces code organization by options, but in some cases it makes more sense to organize code by logical concerns.

2. Lack of a clean and cost-free mechanism for extracting and reusing logic between multiple components. (More details in [Logic Extraction and Reuse](#logic-extraction-and-reuse))

The APIs proposed in this RFC provide the users with more flexibility when organizing component code. Instead of being forced to always organize code by options, code can now be organized as functions each dealing with a specific feature. The APIs also make it more straightforward to extract and reuse logic between components, or even outside components. We will show how these goals are achieved in the [Detailed Design](#detailed-design) section.

### Better Type Inference

Another common feature request from developers working on large projects is better TypeScript support. Vue's current API has posed some challenges when it comes to integration with TypeScript, mostly due to the fact that Vue relies on a single `this` context for exposing properties, and that the use of `this` in a Vue component is a bit more magical than plain JavaScript (e.g. `this` inside functions nested under `methods` points to the component instance rather than the `methods` object). In other words, Vue's existing API simply wasn't designed with type inference in mind, and that creates a lot of complexity when trying to make it work nicely with TypeScript.

Most users who use Vue with TypeScript today are using `vue-class-component`, a library that allows components to be authored as TypeScript classes (with the help of decorators). While designing 3.0, we have attempted to provide a built-in Class API to better tackle the typing issues in a [previous (dropped) RFC](https://github.com/vuejs/rfcs/pull/17). However, as we discussed and iterated on the design, we noticed that in order for the Class API to resolve the typing issues, it must rely on decorators - which is a very unstable stage 2 proposal with a lot of uncertainty regarding its implementation details. This makes it a rather risky foundation to build upon. (More details on Class API type issues [here](#type-issues-with-class-api))

In comparison, the APIs proposed in this RFC utilize mostly plain variables and functions, which are naturally type friendly. Code written with the proposed APIs can enjoy full type inference with little need for manual type hints. This also means that code written with the proposed APIs will look almost identical in TypeScript and plain JavaScript, so even non-TypeScript users will potentially be able to benefit from the typings for better IDE support.

## Detailed Design

### API Introduction

Instead of bringing in new concepts, the APIs being proposed here are more about exposing Vue's core capabilities - such as creating and observing reactive state - as standalone functions. Here we will introduce a number of the most fundamental APIs and how they can be used in place of 2.x options to express in-component logic. Note this section focuses on introducing the basic ideas so it does not goes into full details for each API. Full API specs can be found in the [API Reference](https://vue-composition-api-rfc.netlify.com/api.html) section.

#### Reactive State and Side Effects

Let's start with a simple task: declaring some reactive state.

``` js
import { reactive } from 'vue'

// reactive state
const state = reactive({
  count: 0
})
```

`reactive` is the equivalent of the current `Vue.observable()` API in 2.x, renamed to avoid confusion with RxJS observables. Here, the returned `state` is a reactive object that all Vue users should be familiar with.

The essential use case for reactive state in Vue is that we can use it during render. Thanks to dependency tracking, the view automatically updates when reactive state changes. Rendering something in the DOM is considered a "side effect": our program is modifying state external to the program itself (the DOM). To apply and *automatically re-apply* a side effect based on reactive state, we can use the `watch` API:

``` js
import { reactive, watch } from 'vue'

const state = reactive({
  count: 0
})

watch(() => {
  document.body.innerHTML = `count is ${state.count}`
})
```

`watch` expects a function that applies the desired side effect (in this case, setting `innerHTML`). It executes the function immediately, and tracks all the reactive state properties it used during the execution as dependencies. Here, `state.count` would be tracked as a dependency for this watcher after the initial execution. When `state.count` is mutated at a future time, the inner function will be executed again.

This is the very essence of Vue's reactivity system. When you return an object from `data()` in a component, it is internally made reactive by `reactive()`. The template is compiled into a render function (think of it as a more efficient `innerHTML`) that makes use of these reactive properties.

Continuing the above example, this is how we would handle user input:

``` js
function increment() {
  state.count++
}

document.body.addEventListener('click', increment)
```

But with Vue's templating system we don't need to wrangle with `innerHTML` or manually attaching event listeners. Let's simplify the example with a hypothetical `renderTemplate` method so we can focus on the reactivity side:

``` js
import { reactive, watch } from 'vue'

const state = reactive({
  count: 0
})

function increment() {
  state.count++
}

const renderContext = {
  state,
  increment
}

watch(() => {
  // hypothetical internal code, NOT actual API
  renderTemplate(
    `<button @click="increment">{{ state.count }}</button>`,
    renderContext
  )
})
```

#### Computed State and Refs

Sometimes we need state that depends on other state - in Vue this is handled with *computed properties*. To directly create a computed value, we can use the `computed` API:

``` js
import { reactive, computed } from 'vue'

const state = reactive({
  count: 0
})

const double = computed(() => state.count * 2)
```

What is `computed` returning here? If we take a guess at how `computed` is implemented internally, we might come up with something like this:

``` js
// simplified pseudo code
function computed(getter) {
  let value
  watch(() => {
    value = getter()
  })
  return value
}
```

But we know this won't work: if `value` is a primitive type like `number`, its connection to the update logic inside `computed` will be lost once it's returned. This is because JavaScript primitive types are passed by value, not by reference:

![pass by value vs pass by reference](https://www.mathwarehouse.com/programming/images/pass-by-reference-vs-pass-by-value-animation.gif)

The same problem would occur when a value is assigned to an object as a property. A reactive value wouldn't be very useful if it cannot retain its reactivity when assigned as a property or returned from functions. In order to make sure we can always read the latest value of a computation, we need to wrap the actual value in an object and return that object instead:

``` js
// simplified pseudo code
function computed(getter) {
  const ref = {
    value: null
  }
  watch(() => {
    ref.value = getter()
  })
  return ref
}
```

In addition, we also need to intercept read / write operations to the object's `.value` property to perform dependency tracking and change notification (code omitted here for simplicity). Now we can pass the computed value around by reference, without worrying about losing reactivity. The trade-off is that in order to retrieve the latest value, we now need to access it via `.value`:

``` js
const double = computed(() => state.count * 2)

watch(() => {
  console.log(double.value)
}) // -> 0

state.count++ // -> 2
```

**Here `double` is an object that we call a "ref", as it serves as a reactive reference to the internal value it is holding.**

> You might be aware that Vue already has the concept of "refs", but only for referencing DOM elements or component instances in templates ("template refs"). Check out [this](https://vue-composition-api-rfc.netlify.com/api.html#template-refs) to see how the new refs system can be used for both logical state and template refs.

In addition to computed refs, we can also directly create plain mutable refs using the `ref` API:

``` js
const count = ref(0)
console.log(count.value) // 0

count.value++
console.log(count.value) // 1
```

#### Ref Unwrapping

::: v-pre
We can expose a ref as a property on the render context. Internally, Vue will perform special treatment for refs so that when a ref is encountered on the render context, the context directly exposes its inner value. This means in the template, we can directly write `{{ count }}` instead of `{{ count.value }}`.
:::

Here's a version of the same counter example, using `ref` instead of `reactive`:

``` js
import { ref, watch } from 'vue'

const count = ref(0)

function increment() {
  count.value++
}

const renderContext = {
  count,
  increment
}

watch(() => {
  renderTemplate(
    `<button @click="increment">{{ count }}</button>`,
    renderContext
  )
})
```

In addition, when a ref is nested as a property under a reactive object, it is also automatically unwrapped on access:

``` js
const state = reactive({
  count: 0,
  double: computed(() => state.count * 2)
})

// no need to use `state.double.value`
console.log(state.double)
```

#### Usage in Components

Our code so far already provides a working UI that can update based on user input - but the code runs only once and is not reusable. If we want to reuse the logic, a reasonable next step seems to be refactoring it into a function:

``` js
import { reactive, computed, watch } from 'vue'

function setup() {
  const state = reactive({
    count: 0,
    double: computed(() => state.count * 2)
  })

  function increment() {
    state.count++
  }

  return {
    state,
    increment
  }
}

const renderContext = setup()

watch(() => {
  renderTemplate(
    `<button @click="increment">
      Count is: {{ state.count }}, double is: {{ state.double }}
    </button>`,
    renderContext
  )
})
```

> Note how the above code doesn't rely on the presence of a component instance. Indeed, the APIs introduced so far can all be used outside the context of components, allowing us to leverage Vue's reactivity system in a wider range of scenarios.


Now if we leave the tasks of calling `setup()`, creating the watcher, and rendering the template to the framework, we can define a component with just the `setup()` function and the template:

``` html
<template>
  <button @click="increment">
    Count is: {{ state.count }}, double is: {{ state.double }}
  </button>
</template>

<script>
import { reactive, computed } from 'vue'

export default {
  setup() {
    const state = reactive({
      count: 0,
      double: computed(() => state.count * 2)
    })

    function increment() {
      state.count++
    }

    return {
      state,
      increment
    }
  }
}
</script>
```

This is the single-file component format we are familiar with, with only the logical part (`<script>`) expressed in a different format. Template syntax remains exactly the same. `<style>` is omitted, but would also work exactly the same.

#### Lifecycle Hooks

So far we have covered the pure state aspect of a component: reactive state, computed state and mutating state on user input. But a component may also need to perform side effects - for example, logging to the console, sending an ajax request, or setting up an event listener on `window`. These side effects are typically performed at the following timings:

- When some state changes;
- When the component is mounted, updated or unmounted (lifecycle hooks).

We know that we can use the `watch` API to apply side effects based on state changes. As for performing side effects in different lifecycle hooks, we can use the dedicated `onXXX` APIs (which directly mirror the existing lifecycle options):

``` js
import { onMounted } from 'vue'

export default {
  setup() {
    onMounted(() => {
      console.log('component is mounted!')
    })
  }
}
```

These lifecycle registration methods can only be used during the invocation of a `setup` hook. It automatically figures out the current instance calling the `setup` hook using internal global state. It is intentionally designed this way to reduce friction when extracting logic into external functions.

> More details about these APIs can be found in the [API Reference](https://vue-composition-api-rfc.netlify.com/api.html). However, we recommend finishing the following sections before digging into the design details.

### Code Organization

At this point we have replicated the component API with imported functions, but what for? Defining components with options seems to be much more organized than mixing everything together in a big function!

This is an understandable first impression. But as mentioned in the Motivations section, we believe the Composition API actually leads to *better* organized code, particularly in complex components. Here we will try to explain why.

#### What is "Organized Code"?

Let's take a step back and consider what we really mean when we talk about "organized code". The end goal of keeping code organized should be making the code easier to read and understand. And what do we mean by "understanding" the code? Can we really claim that we "understand" a component just because we know what options it contains? Have you ever run into a big component authored by another developer (for example [this one](https://github.com/vuejs/vue-cli/blob/dev/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L198-L404)), and have a hard time wrapping your head around it?

Think about how we would walk a fellow developer through a big component like the one linked above. You will most likely start with "this component is dealing with X, Y and Z" instead of "this component has these data properties, these computed properties and these methods". When it comes to understanding a component, we care more about "what the component is trying to do" (i.e. the intentions behind the code) rather than "what options the component happens to use". While code written with options-based API naturally answers the latter, it does a rather poor job at expressing the former.

#### Logical Concerns vs. Option Types

Let's define the "X, Y and Z" the component is dealing with as **logical concerns**. The readability problem is typically non-present in small, single purpose components because the entire component deals with a single logical concern. However, the issue becomes much more prominent in advanced use cases. Take the [Vue CLI UI file explorer](https://github.com/vuejs/vue-cli/blob/dev/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L198-L404) as an example. The component has to deal with many different logical concerns:

- Tracking current folder state and displaying its content
- Handling folder navigation (opening, closing, refreshing...)
- Handling new folder creation
- Toggling show favorite folders only
- Toggling show hidden folders
- Handling current working directory changes

Can you instantly recognize and tell these logical concerns apart by reading the options-based code? It surely is difficult. You will notice that code related to a specific logical concern is often fragmented and scattered all over the place. For example, the "create new folder" feature makes use of [two data properties](https://github.com/vuejs/vue-cli/blob/dev/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L221-L222), [one computed property](https://github.com/vuejs/vue-cli/blob/dev/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L240), and [a method](https://github.com/vuejs/vue-cli/blob/dev/packages/@vue/cli-ui/src/components/folder/FolderExplorer.vue#L387) - where the method is defined in a location more than a hundred lines away from the data properties.

If we color-code each of these logical concerns, we notice how fragmented they are when expressed with component options:

<p align="center">
  <img src="https://user-images.githubusercontent.com/499550/62783021-7ce24400-ba89-11e9-9dd3-36f4f6b1fae2.png" alt="file explorer (before)" width="131">
</p>

Such fragmentation is exactly what makes it difficult to understand and maintain a complex component. The forced separation via options obscures the underlying logical concerns. In addition, when working on a single logical concern we have to constantly "jump" around option blocks to find the parts related to that concern.

> Note: the original code can probably be improved a few places, but we are showing it off the latest commit (as of this writing) without modification to provide an example of actual in production code we wrote ourselves.

It would be much nicer if we could collocate code related to the same logical concern. And this is exactly what the Composition API enables us to do. The "create new folder" feature can be written this way:

``` js
function useCreateFolder (openFolder) {
  // originally data properties
  const showNewFolder = ref(false)
  const newFolderName = ref('')

  // originally computed property
  const newFolderValid = computed(() => isValidMultiName(newFolderName.value))

  // originally a method
  async function createFolder () {
    if (!newFolderValid.value) return
    const result = await mutate({
      mutation: FOLDER_CREATE,
      variables: {
        name: newFolderName.value
      }
    })
    openFolder(result.data.folderCreate.path)
    newFolderName.value = ''
    showNewFolder.value = false
  }

  return {
    showNewFolder,
    newFolderName,
    newFolderValid,
    createFolder
  }
}
```

Notice how all the logic related to the create new folder feature is now collocated and encapsulated in a single function. The function is also somewhat self-documenting due to its descriptive name. This is what we call a **composition function**. It is a recommended convention to start the function's name with `use` to indicate that it is a composition function. This pattern can be applied to all the other logical concerns in the component, resulting in a number of nicely decoupled functions:

<p align="center">
  <img src="https://user-images.githubusercontent.com/499550/62783026-810e6180-ba89-11e9-8774-e7771c8095d6.png" alt="file explorer (comparison)" width="600">
</p>

> This comparison excludes import statements and the `setup()` function. The full component re-implemented using the Composition API can be found [here](https://gist.github.com/yyx990803/8854f8f6a97631576c14b63c8acd8f2e).

Code for each logical concern is now collocated together in a composition function. This greatly reduces the need for constant "jumps" when working on a large component. Composition functions can also be folded in the editor to make the component much easier to scan:

``` js
export default {
  setup() { // ...
  }
}

function useCurrentFolderData(networkState) { // ...
}

function useFolderNavigation({ networkState, currentFolderData }) { // ...
}

function useFavoriteFolder(currentFolderData) { // ...
}

function useHiddenFolders() { // ...
}

function useCreateFolder(openFolder) { // ...
}
```

The `setup()` function now primarily serves as an entry point where all the composition functions are invoked:

``` js
export default {
  setup () {
    // Network
    const { networkState } = useNetworkState()

    // Folder
    const { folders, currentFolderData } = useCurrentFolderData(networkState)
    const folderNavigation = useFolderNavigation({ networkState, currentFolderData })
    const { favoriteFolders, toggleFavorite } = useFavoriteFolders(currentFolderData)
    const { showHiddenFolders } = useHiddenFolders()
    const createFolder = useCreateFolder(folderNavigation.openFolder)

    // Current working directory
    resetCwdOnLeave()
    const { updateOnCwdChanged } = useCwdUtils()

    // Utils
    const { slicePath } = usePathUtils()

    return {
      networkState,
      folders,
      currentFolderData,
      folderNavigation,
      favoriteFolders,
      toggleFavorite,
      showHiddenFolders,
      createFolder,
      updateOnCwdChanged,
      slicePath
    }
  }
}
```

Granted, this is code that we didn't have to write when using the options API. But note how the `setup` function almost reads like a verbal description of what the component is trying to do - this is information that was totally missing in the options-based version. You can also clearly see the dependency flow between the composition functions based on the arguments being passed around. Finally, the return statement serves as the single place to check what is exposed to the template.

Given the same functionality, a component defined via options and a component defined via composition functions manifest two different ways of expressing the same underlying logic. Options-based API forces us to organize code based on *option types*, while the Composition API enables us to organize code based on *logical concerns*.

### Logic Extraction and Reuse

The Composition API is extremely flexible when it comes to extracting and reusing logic across components. Instead of relying on the magical `this` context, a composition function relies only on its arguments and globally imported Vue APIs. You can reuse any part of your component logic by simply exporting it as a function. You can even achieve the equivalent of `extends` by exporting the entire `setup` function of a component.

Let's check out an example: tracking the mouse position.

``` js
import { ref, onMounted, onUnmounted } from 'vue'

export function useMousePosition() {
  const x = ref(0)
  const y = ref(0)

  function update(e) {
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
```

This is how a component can make use of the function:

``` js
import { useMousePosition } from './mouse'

export default {
  setup() {
    const { x, y } = useMousePosition()
    // other logic...
    return { x, y }
  }
}
```

In the Composition API version of the file explorer example, we have extracted some utility code (such as `usePathUtils` and `useCwdUtils`) into external files because we found them useful to other components.

Similar logic reuse can also be achieved using existing patterns such as mixins, higher-order components or renderless components (via scoped slots). There are plenty of information explaining these patterns on the internet, so we shall not repeat them in full details here. The high level idea is that each of these patterns has respective drawbacks when compared to composition functions:

- Unclear sources for properties exposed on the render context. For example, when reading the template of a component using multiple mixins, it can be difficult to tell from which mixin a specific property was injected from.

- Namespace clashing. Mixins can potentially clash on property and method names, while HOCs can clash on expected prop names.

- Performance. HOCs and renderless components require extra stateful component instances that come at a performance cost.

In comparison, with Composition API:

- Properties exposed to the template have clear sources since they are values returned from composition functions.

- Returned values from composition functions can be arbitrarily named so there is no namespace collision.

- There are no unnecessary component instances created just for logic reuse.

### Usage Alongside Existing API

The Composition API can be used alongside the existing options-based API.

- The Composition API is resolved before 2.x options (`data`, `computed` & `methods`) and will have no access to the properties defined by those options.

- Properties returned from `setup()` will be exposed on `this` and will be accessible inside 2.x options.

### Plugin Development

Many Vue plugins today inject properties onto `this`. For example, Vue Router injects `this.$route` and `this.$router`, and Vuex injects `this.$store`. This has made type inference tricky since each plugin requires the user to augment the Vue typing for injected properties.

When using the Composition API, there is no `this`. Instead, plugins will leverage [`provide` and `inject`](https://vue-composition-api-rfc.netlify.com/api.html#provide-inject) internally and expose a composition function. The following is hypothetical code for a plugin:

``` js
const StoreSymbol = Symbol()

export function provideStore(store) {
  provide(StoreSymbol, store)
}

export function useStore() {
  const store = inject(StoreSymbol)
  if (!store) {
    // throw error, no store provided
  }
  return store
}
```

And in consuming code:

``` js
// provide store at component root
//
const App = {
  setup() {
    provideStore(store)
  }
}

const Child = {
  setup() {
    const store = useStore()
    // use the store
  }
}
```

Note the store can also be provided via the app-level provide proposed in the [Global API change RFC](https://github.com/vuejs/rfcs/blob/global-api-change/active-rfcs/0000-global-api-change.md#provide--inject), but the `useStore` style API in the consuming component would be the same.

## Drawbacks

### Overhead of Introducing Refs

Ref is technically the only "new" concept introduced in this proposal. It is introduced in order to pass reactive values around as variables without relying on access to `this`. The drawbacks are:

1. When using the Composition API, we will need to constantly distinguish refs from plain values and objects, increasing the mental burden when working with the API.

    The mental burden can be greatly reduced by using a naming convention (e.g. suffixing all ref variables as `xxxRef`), or by using a type system. On the other hand, due to the improved flexibility in code organization, component logic will more often be isolated into small functions where the local context is simple and the overhead of refs are easily manageable.

2. Reading and mutating refs are more verbose than working with plain values due to the need for `.value`.

    Some have suggested compile-time syntax sugar (similar to Svelte 3) to solve this. While it is technically feasible, we do not believe it would make sense as the default for Vue (as discussed in [Comparison with Svelte](#comparison-with-svelte)). That said, this is technically feasible in userland as a Babel plugin.

We have discussed whether it is possible to completely avoid the Ref concept and use only reactive objects, however:

- Computed getters can return primitive types, so a Ref-like container is unavoidable.

- Composition functions expecting or returning only primitive types also need to wrap the value in an object just for reactivity's sake. It's very likely that users will end up inventing their own Ref like patterns (and causing ecosystem fragmentation) if there is not a standard implementation provided by the framework.

### Ref vs. Reactive

Understandably, users may get confused regarding which to use between `ref` and `reactive`. First thing to know is that you **will** need to understand both to efficiently make use of the Composition API. Using one exclusively will most likely lead to esoteric workarounds or reinvented wheels.

The difference between using `ref` and `reactive` can be somewhat compared to how you would write standard JavaScript logic:

``` js
// style 1: separate variables
let x = 0
let y = 0

function updatePosition(e) {
  x = e.pageX
  y = e.pageY
}

// --- compared to ---

// style 2: single object
const pos = {
  x: 0,
  y: 0
}

function updatePosition(e) {
  pos.x = e.pageX
  pos.y = e.pageY
}
```

- If using `ref`, we are largely translating style (1) to a more verbose equivalent using refs (in order to make the primitive values reactive).

- Using `reactive` is nearly identical to style (2). We only need to create the object with `reactive` and that's it.

However, the problem with going `reactive`-only is that the consumer of a composition function must keep the reference to the returned object at all times in order to retain reactivity. The object cannot be destructured or spread:

``` js
// composition function
function useMousePosition() {
  const pos = reactive({
    x: 0,
    y: 0
  })

  // ...
  return pos
}

// consuming component
export default {
  setup() {
    // reactivity lost!
    const { x, y } = useMousePosition()
    return {
      x,
      y
    }

    // reactivity lost!
    return {
      ...useMousePosition()
    }

    // this is the only way to retain reactivity.
    // you must return `pos` as-is and reference x and y as `pos.x` and `pos.y`
    // in the template.
    return {
      pos: useMousePosition()
    }
  }
}
```

The [`toRefs`](https://vue-composition-api-rfc.netlify.com/api.html#torefs) API is provided to deal with this constraint - it converts each property on a reactive object to a corresponding ref:

``` js
function useMousePosition() {
  const pos = reactive({
    x: 0,
    y: 0
  })

  // ...
  return toRefs(pos)
}

// x & y are now refs!
const { x, y } = useMousePosition()
```

To sum up, there are two viable styles:

1. Use `ref` and `reactive` just like how you'd declare primitive type variables and object variables in normal JavaScript. It is recommended to use a type system with IDE support when using this style.

2. Use `reactive` whenever you can, and remember to use `toRefs` when returning reactive objects from composition functions. This reduces the mental overhead of refs but does not eliminate the need to be familiar with the concept.

At this stage, we believe it is too early to mandate a best practice on `ref` vs. `reactive`. We recommend you to go with the style that aligns with your mental model better from the two options above. We will be collecting real world user feedback and eventually provide more definitive guidance on this topic.

### Verbosity of the Return Statement

Some users have raised concerns about the return statement in `setup()` being verbose and feeling like boilerplate.

We believe an explicit return statement is beneficial for maintainability. It gives us the ability to explicitly control what gets exposed to the template, and serves as the starting point when tracing where a template property is defined in a component.

There were suggestions to automatically expose variables declared in `setup()`, making the return statement optional. Again, we don't think this should be the default since it would go against the intuition of standard JavaScript. However, there are possible ways to make it less of a chore in userland:

- IDE extension that automatically generates the return statement based on variables declared in `setup()`

- Babel plugin that implicitly generates and inserts the return statement.

### More Flexibility Requires More Discipline

Many users have pointed out that while the Composition API provides more flexibility in code organization, it also requires more discipline from the developer to "do it right". Some worry that the API will lead to spaghetti code in inexperienced hands. In other words, while the Composition API raises the upper bound of code quality, it also lowers the lower bound.

We agree with that to a certain extent. However, we believe that:

1. The gain in the upper bound far outweighs the loss in the lower bound.

2. We can effectively address the code organization problem with proper documentation and community guidance.

Some users used Angular 1 controllers as examples of how the design could lead to poorly written code. The biggest difference between the Composition API and Angular 1 controllers is that it doesn't rely on a shared scope context. This makes it significantly easier to split out logic into separate functions, which is the core mechanism of JavaScript code organization.

Any JavaScript program starts with an entry file (think of it as the `setup()` for a program). We organize the program by splitting it into functions and modules based on logical concerns. **The Composition API enables us to do the same for Vue component code.** In other words, skills in writing well-organized JavaScript code translates directly into skills of writing well-organized Vue code when using the Composition API.

## Adoption strategy

The Composition API is purely additive and does not affect / deprecate any existing 2.x APIs. It has been made available as a 2.x plugin via the [`@vue/composition` library](https://github.com/vuejs/composition-api/). The library's primary goal is to provide a way to experiment with the API and to collect feedback. The current implementation is up-to-date with this proposal, but may contain minor inconsistencies due to technical constraints of being a plugin. It may also receive breaking changes as this proposal is updated, so we do not recommend using it in production at this stage.

We intend to ship the API as built-in in 3.0. It will be usable alongside existing 2.x options.

For users who opt to use the Composition API exclusively in an app, it is possible to provide a compile-time flag to drop code only used for 2.x options and reduce the library size. However, this is completely optional.

The API will be positioned as an advanced feature, since the problems it aims to address appear primarily in large scale applications. We do not intend to overhaul the documentation to use it as the default. Instead, it will have its own dedicated section in the docs.

## Appendix

### Type Issues with Class API

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

### Comparison with React Hooks

The function based API provides the same level of logic composition capabilities as React Hooks, but with some important differences. Unlike React hooks, the `setup()` function is called only once. This means code using Vue's Composition API is:

- In general more aligned with the intuitions of idiomatic JavaScript code;
- Not sensitive to call order and can be conditional;
- Not called repeatedly on each render and produces less GC pressure;
- Not subject to the issue where `useCallback` is almost always needed in order to prevent inline handlers causing over-re-rendering of child components;
- Not subject to the issue where `useEffect` and `useMemo` may capture stale variables if the user forgets to pass the correct dependency array. Vue's automated dependency tracking ensures watchers and computed values are always correctly invalidated.

We acknowledge the creativity of React Hooks, and it is a major source of inspiration for this proposal. However, the issues mentioned above do exist in its design and we noticed Vue's reactivity model happens to provide a way around them.

### Comparison with Svelte

Although taking very different routes, the Composition API and Svelte 3's compiler-based approach actually shares quite a bit in common conceptually. Here's a side-by-side example:

#### Vue

``` html
<script>
import { ref, watch, onMounted } from 'vue'

export default {
  setup() {
    const count = ref(0)

    function increment() {
      count.value++
    }

    watch(() => console.log(count.value))

    onMounted(() => console.log('mounted!'))

    return {
      count,
      increment
    }
  }
}
</script>
```

#### Svelte

``` html
<script>
import { onMount } from 'svelte'

let count = 0

function increment() {
  count++
}

$: console.log(count)

onMount(() => console.log('mounted!'))
</script>
```

Svelte code looks more concise because it does the following at compile time:

- Implicitly wraps the entire `<script>` block (except import statements) into a function that is called for each component instance (instead of being executed only once)
- Implicitly registers reactivity on variable mutations
- Implicitly exposes all in-scope variables to the render context
- Compiles `$` statements into re-executed code

Technically, we can do the same in Vue (and it's possible via userland Babel plugins). The main reason we are not doing it is **alignment with standard JavaScript**. If you extract the code from the `<script>` block of a Vue file, we want it to work exactly the same as a standard ES module. The code inside a Svelte `<script>` block, on the other hand, is technically no longer standard JavaScript. There are a number of problems we see with this compiler-based approach:

1. Code works differently with/without compilation. As a progressive framework, many Vue users may wish/need/have to use it without a build setup, so the compiled version cannot be the default. Svelte, on the other hand, positions itself as a compiler and can *only* be used with a build step. This is a trade-off both frameworks are making consciously.

2. Code works differently inside/outside components. When trying to extract logic out of a Svelte component and into standard JavaScript files, we will lose the magical concise syntax and have to fall back to a [more verbose lower-level API](https://svelte.dev/docs#svelte_store).

3. Svelte's reactivity compilation only works for top-level variables - it doesn't touch variables declared inside functions, so we [cannot encapsulate reactive state in a function declared inside a component](https://svelte.dev/repl/4b000d682c0548e79697ddffaeb757a3?version=3.6.2). This places non-trivial constraints on code organization with functions - which, as we have demonstrated in this RFC, is important for keeping large components maintainable.

4. [Non-standard semantics makes it problematic to integrate with TypeScript](https://github.com/sveltejs/svelte/issues/1639).

This is in no way saying that Svelte 3 is bad idea - in fact, it's a very innovative approach and we highly respect Rich's work. But based on Vue's design constraints and goals, we have to make different trade-offs.
