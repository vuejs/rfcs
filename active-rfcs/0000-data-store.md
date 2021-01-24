- Start Date: 2021-01-24
- Target Major Version: 3.4.9
- Reference Issues: https://github.com/vuejs/vue-router/issues/3452
- Implementation PR: (leave this empty)

# Summary
Maintain a store by history state key. Can be used to save something.


# Basic example

[Online Demo](https://hezedu.github.io/SomethingBoring/vue-router-positionstore-memory-leak/pr-demo.html#/)
# Motivation

Unlike [window.history](https://developer.mozilla.org/en-US/docs/Web/API/History), vue-router has no API to change history.state.

But sometimes we need it to store some unique keys, like the `positionStore`. <br>
So I replaced `positionStore` with `dataStore`. This way you can store more things. <br>

# Detailed design

<b>Fixed:</b> [#3445: vue-router positionStore memory leak](https://github.com/vuejs/vue-router/issues/3445)

<b>reconstruct:</b> `util/state-key`

## New API
### router.setData(key, data)
  - `key` ___string___
  - `data` ___any___
### router.getData(key)
  - `key` ___string___ 
  - Returns: ___any___
### router.removeData(key)
  - `key` ___string___



# Adoption strategy
Expose three new APIs, And there is no need to develop `state` related APIs in the future.


