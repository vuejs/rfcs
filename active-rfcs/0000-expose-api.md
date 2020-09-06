- Start Date: 2020-09-06
- Target Major Version: 3.x
- Reference Issues: (no)
- Implementation PR: (leave this empty)

# Summary

Provide one more composition API like `expose({ ...publicMembers })` so component authors could use that API in `setup()` to clearly control what would be explicitly exposed to the component instance publicly, which could be accessed by the parent as a ref.

# Basic example

```javascript
import { ref, expose } from 'vue'

export default {
  setup() {
    const count = ref(0)
    function increment() {
      count.value++
    }
    
    // the exposed object will be what is exposed to
    // the parent that obtained a ref to this component
    expose({
      increment
    })

    return () => JSX
  }
}
```

```javascript
import { ref, expose } from 'vue'

export default {
  setup() {
    const count = ref(0)
    function increment() {
      count.value++
    }
    
    // the exposed object will be what is exposed to
    // the parent that obtained a ref to this component
    expose({
      increment
    })

    return { increment, count }
  }
}
```

Here in the both 2 cases, the parent could only access `increment` member from this component ref.

# Motivation

Currently, from parent as a ref, we could:

- access all the returned members in `setup`. or
- no members if the `setup` returns a render function.

But actually, all the returned members in `setup` is supposed to be used in template. Therefore:

1. The component which is returning a render function from `setup` couldn't expose members to public.
2. Public members are just implicitly exposed without a clear API, which may confuse people (we didn't mention this on our docs or RFCs)
3. Members in public couldn't differentiate to those in template, which is supposed to be inconvenient in some strict cases to protect private members which is used in template

So here I suggest adding one more composition API `expose(obj)` to do this explicitly. And it also supports people returning a render function in `setup()`.

# Detailed design

Add a new function `expose` from `vue` package.

```js
import { expose } from 'vue'
```

This function could take an object argument that each member in it would be a public member of this component instance.

Additionally, to avoid breaking change, if the `expose()` function wasn't called in `setup()`, then the public members would be just as before as well.

For example:

```javascript
import { ref, expose } from 'vue'

export default {
  setup() {
    const count = ref(0)
    function increment() {
      count.value++
    }

    return { increment, count }
  }
}
```

Without calling `expose()`, the parent could access both `increment` and `count` as normal.

# Drawbacks

Nothing found so far. Because it's just a explicit way from another implicit way.

# Alternatives

no

# Adoption strategy

Additionally call this API explicitly to make the public members more clear and easy to understand and maintain.

# Unresolved questions

Nothing found so far.
