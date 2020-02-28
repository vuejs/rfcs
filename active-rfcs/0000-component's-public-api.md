- Start Date: 2020-02-27
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

Introduce a new `expose` option to declare component's public API available for parent components via `refs`. Restrict access to component's context by default.

# Basic example

At the moment you can access the whole component's context without any limitations when using `refs`.

```html
<template>
  <input ref="input">
</template>

<script>
  export default {
    name: 'MyInput',
    methods: {
      focus() { this.$refs.input.focus() }
    }
  }
</script>
```

```html
<template>
  <MyInput ref="input" />
</template>

<script>
  export default {
    name: 'Parent',
    mounted() {
      this.$refs.input.focus() // you can access anything on the context
    }
  }
</script>
```

After the change you'll be required to expose your data and methods explicitly.

```html
<template>
  <input ref="input">
</template>

<script>
  export default {
    name: 'MyInput',
    expose: ['focus'],
    methods: {
      focus() { this.$refs.input.focus() },
      blur() { this.$refs.input.blur() },
    }
  }
</script>
```

```html
<template>
  <MyInput ref="input" />
</template>

<script>
  export default {
    name: 'Parent',
    mounted() {
      this.$refs.input.focus() // exposed
      this.$refs.input.blue() // error, not exposed
    }
  }
</script>
```

# Motivation

Right now component's context is free to access by anyone and that brings up a number of issues:

1. You can not guarantee consistent component's behaviour due to external modifications. You can modify components data and won't be able to tell where that change came from.
2. There's no contract between receiver and provider components. This could lead to refactoring problems when there's an unused method within the component, but it's required by another component and there's no easy way to tell that.
3. No clear separation of data that is required by the component itself and other components.

To fix these issues components would be required to explicitly declare their public interface.

# Detailed design

Components should not be able to directly access other components context anymore. To declare component's public interface authors must use a new `expose` options and describe the data that should be exposed.

## `expose` option

A several `expose` configurations should be supported.

### Array of strings

```html
<script>
  export default {
    expose: ['foo', 'bar', 'baz'],
    data() {
      return {
        foo: null
      }
    },
    methods: {
      bar() {},
    },
    computed: {
      baz() {}
    }
  }
</script>
```

### Object syntax

```html
<script>
  export const BAR_SYMBOL = Symbol()

  export default {
    expose: {
      foo: 'bar', // expose 'foo' as 'bar'
      bar: BAR_SYMBOL // expose 'bar' as a BAR_SYMBOL
    },
    data() {
      return {
        foo: null
      }
    },
    methods: {
      bar() {}
    }
  }
</script>
```

### Function

Functional configuration is more complicated because you can easily loose reactivity there.

```html
<script>
  export default {
    expose() {
      const { foo, bar } = this
      return {
        foo, // not reactive
        bar
      }
    },
    data() {
      return {
        foo: null
      }
    },
    methods: {
      bar() {},
    }
  }
</script>
```

To solve this issue you could either go with the Composition API or wrap your data inside an object.

```html
<script>
  export default {
    expose() {
      const { foo, bar } = this
      return {
        foo, // foo's value is reactive
        bar
      }
    },
    data() {
      return {
        foo: {
          value: null
        }
      }
    },
    methods: {
      bar() {},
    }
  }
</script>
```

**Composition API example**

```html
<script>
  import { ref } from 'vue'

  export default {
    name: 'MyComponent',
    setup() {
      const foo = ref(null)
      const bar = () => {}
      return {
        foo,
        bar
      }
    },
    expose() {
      const { foo, bar } = this
      return {
        foo, // reactive
        bar
      }
    },
  }
</script>
```

The exposed interface above could be utilized as following:

```html
<template>
  <MyComponent :ref="myComponent" />
</template>

<script>
  export default {
    mounted() {
      console.log(this.$refs.myComponent.foo.value)
      this.$refs.myComponent.bar()
    }
  }
</script>
```

To expose refs use `mounted` hook and object as a wrapper to preserve reactivity.

```html
<template>
  <input ref="input">
</template>

<script>
  export const INPUT_EXPOSED = Symbol()
  
  export default {
    expose() {
      const { exposed } = this
      return {
        [INPUT_EXPOSED]: exposed
      }
    },
    data() {
      return {
        exposed: {
          inputRef: null
        }
      }
    },
    mounted() {
      this.exposed.inputRef = this.$refs.input
    }
  }
</script>
```

Or function refs:

```html
<template>
  <input :ref="(input) => exposed.inputRef = input">
</template>

<script>
  export const INPUT_EXPOSED = Symbol()
  
  export default {
    expose() {
      const { exposed } = this
      return {
        [INPUT_EXPOSED]: exposed
      }
    },
    data() {
      return {
        exposed: {
          inputRef: null
        }
      }
    },
  }
</script>
```

Alternative way of accessing refs:

```html
<template>
  <input ref="input">
</template>

<script>
  export const GET_INPUT_REF = Symbol()
  
  export default {
    name: 'MyInput',
    expose() {
      return {
        [GET_INPUT_REF]: () => {
          return this.$refs.input
        }
      }
    },
  }
</script>
```

```html
<template>
  <MyInput ref="input">
</template>

<script>
  import { GET_INPUT_REF } from 'MyInput.vue'
  
  export default {
    mounted() {
      console.log(this.$refs.input[GET_INPUT_REF]())
    }
  }
</script>
```

# Drawbacks

* `expose` should be executed after data init, `refs` then would require a lot of fiddling around to preserve reactivity (wrapping exposed values with an object at a minimum)
* Could be difficult to implement
* Not backwards compatible (but could only warn in compatibility build for example)

# Alternatives

Using events to expose your component's public interface:  
  ```html
  <template>
    <input ref="input">
  </template>

  <script>    
    export default {
      mounted() {
        const exposed = {
          focusInput: () => {
            this.$refs.input.focus()
          }
        };
        this.$emit('ready', exposed);
      }
    }
  </script>
  ```

  **Pros**:

  * Does not require any API change
  * Easy to implement
  * Fixes `refs` issue

  **Cons**:

  * Can not be enforced by the framework (unless context is always unavailable in refs)
  * Forces to be extra careful with context
  * Creates an unnecessary data flow


# Adoption strategy

Components containing any data required by the parent component via `refs` would be required to explicitly declare their public interface.

# Unresolved questions

* Should it work with mixins?
  
  **Pros**:

  * Clear contract between mixins and component

  **Cons**:

  * Would require a lot of changes if you use mixins extensively
  * Rises entry barrier for mixin usage
  * Does not solve other issues with mixins (implicit props, extending context)
  * Could be complicated to implement (mixin can extend context but should only access exposed parts of it)
  * Unclear whether mixins should be able to extend the `expose` property

* Should it work with `$parent` access?

* Should it work in tests?
  
  **Pros**:

  * Explicit testing interface
  * Tests for `expose` interface specifically

  **Cons**:

  * Could lead to exposing too much data just for testing which defeats the purpose