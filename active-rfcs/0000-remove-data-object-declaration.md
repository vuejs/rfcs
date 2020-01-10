- Start Date: 2020-01-10
- Target Major Version: 3.x
- Implementation PR: (leave this empty)

# Summary

`data` component option accepts two types of declaration: `function` or `object`. The most common is the `function` declaration, because it creates a new state for each instance of the component. `object` declaration on the other hand shares state between all the component's instances. This RFC focuses on removing `object` declaration for `data`.

# Motivation

There are little or no use-cases for component's instances to have a shared state. Even if you encounter such a case it can be done using `function` declaration.
Having two types of declaration is not novice-friendly and is confusing without proper examples (which are nonexistent in the current documentation).
With a single type of declaration you could achieve the same results and eliminate the confusing part of it.

# Detailed design

`object` declaration should no longer be valid and produce an error with an explanation that only `function` declaration is valid for a component. It should also contain a link to an API and migration example.

Before, using `object` declaration:

```html
<script>
export default {
  data: {
    counter: 1
  },
  methods: {
    increment() {
      this.counter++
    }
  }
}
</script>
```

After, using `function` declaration.

```html
<script>
const sharedObject = { counter: 1 }

export default {
  data() {
    return {
      sharedObject
    }
  },
  methods: {
    increment() {
      this.sharedObject.counter++
    }
  }
}
</script>
```

`sharedObject` will be the same in each component's instance.

# Drawbacks

Users that work with `object` declaration would require migrating to `function` declaration.

It would also not be possible to have top-level shared state.

Examples using `object` declaration should be rewritten using `function` declaration.

# Adoption strategy

Since it's not a common pattern in components migration should be fairly easy.

Migration itself is straightforward:

* Extract shared data into an external object and use it as a property in `data`.
* Rewrite references to the shared data to point to a new shared object.

API page should contain an info block that `object` declaration is deprecated with a guide on how to migrate.

An adapter could be provided to retain backwards compatibility.
