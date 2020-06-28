- Start Date: 2020-06-28
- Target Major Version: 3.0 (but could also applied to 2.X)
- Reference Issues: /
- Implementation PR:

# Summary

Composition API : Setup component with a generator function

# Basic example

```ts
// feature.js
export function useFeature() {
  const state = reactive({
    id: 0,
  })
  const setId = (id) => (state.id = id)

  return {
    id: state.id,
    setId,
  }
}

function* setup() {
  const { id, setId } = useFeature()
  yield { id, setId }

  yield { name: ref('foo') }
  // or
  return { name: ref('foo') }
}
```

This should result to Template Render Context (TRC) :

```js
{
  id: 0,
  setId() { ... },
  name: 'foo'
}
```

# Motivation

Generator function adds like a "incremental" returning values which could be sought with `next().value` method.

Something in the code above should probably tick in your mind: **my feature fields are not returned at the end as we should, but are returned anyway**.

This RFC is linked to [a TC39 discussion](https://es.discourse.group/t/yield-destructured-variables/379) where I had propose to improve the usage of `yield`.

If TC39 accept my proposal, we could have something like that :

```ts
function* setup() {
  yield ({ id, setId } = useFeatureA())

  // another feature with renaming
  yield ({ id: featBId, setId: featBSetId } = useFeatureB())

  return { name: ref('foo') }
}
```

# Detailed design

If you don't understand what it's added to the developer, you have to know that with generator function, we could _export_ fields immediatly without to return it at the end of the function.

Instead of getting one return, _we could have many_. Not really, but you understand now.

Interpreting a generator `setup` function could be slightly the same as a normal `setup` function.
Difference is that the TRC have to be build in several steps.

```ts
function isGenerator(fn) {
  return fn.constructor.name === 'GeneratorFunction'
}

function getSetupResult(setupFn) {
  let setupResult = setupFn(/*...*/)
  if (isGenerator(setupFn)) {
    let yieldValue
    while ((yieldValue = setupFn().next().value)) {
      setupResult = {
        ...setupResult,
        ...yieldValue,
      }
    }
  }
  return Object.freeze(setupResult)
}
```

# Drawbacks

- We don't have summary of Template Render Function
- Two ways for creating a Setup function
- Generator functions are not mastered by all developers (but should to!)

# Adoption strategy

If we implement this proposal, how will existing Vue developers adopt it? Is
this a breaking change? Can we write a codemod? Can we provide a runtime adapter library for the original API it replaces? How will this affect other projects in the Vue ecosystem?

# Unresolved questions

- Typings are lost ? Are they needed in the setup function ?
