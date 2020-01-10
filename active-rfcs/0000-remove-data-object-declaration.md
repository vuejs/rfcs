- Start Date: 2020-01-10
- Target Major Version: 3.x
- Implementation PR: (leave this empty)

# Summary

`data` option accepts two types of declaration: `function` or `object`. The most common is the `function` declaration, because it creates a new state for each instance of the component. `object` declaration on the other hand shares state between all the  instances and works only on a root instance. This RFC primarily focuses on removing `object` declaration for `data`.

# Motivation

There are little or no use-cases for root instances to have a shared state. Even if you encounter such a case it can be achieved using `function` declaration.
Having two types of declaration is not novice-friendly and is confusing without proper examples (which are nonexistent in the current documentation). The additional restriction that `object` declaration is possible only on the root instance is also confusing.
With a unified declaration you could achieve the same result and eliminate the confusing part of it.

# Detailed design

`object` declaration should no longer be valid and produce an error with an explanation that only `function` declaration is valid for `data` on the root instance. It should also contain a link to an API and migration example.

Before, using `object` declaration:

```js
import { createApp, h } from 'vue'

createApp().mount({
  data: {
    counter: 1,
  },
  render() {
    return [
      h('span', this.counter),
      h('button', {
        onClick: () => { this.counter++ }
      }),
    ]
  },
}, '#app')
```

After, using `function` declaration.

```js
import { createApp, h } from 'vue'

createApp().mount({
  data() {
    return {
      counter: 1,
    }
  },
  render() {
    return [
      h('span', this.counter),
      h('button', {
        onClick: () => { this.counter++ }
      }),
    ]
  },
}, '#app')
```

# Drawbacks

Users that work with `object` declaration would require migrating to `function` declaration.

It would also not be possible to have top-level shared state.

Examples using `object` declaration should be rewritten using `function` declaration.

# Adoption strategy

Since it's not a common pattern to have an `object` declaration in root instance migration should be fairly easy.

Migration itself is straightforward:

* Extract shared data into an external object and use it as a property in `data`.
* Rewrite references to the shared data to point to a new shared object.

API page should contain an info block that `object` declaration is deprecated with a guide on how to migrate.

An adapter could be provided to retain backwards compatibility.
