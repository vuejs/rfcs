- Start Date: 2020-02-27
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

Use `provide` to define component's public API available for parent components via `refs`.

# Basic example

Right now you can access the whole component's context without any limitations when using `refs`.

```html
<template>
  <div />
</template>

<script>
  export default {
    name: 'Foo',
    methods: {
      someMethod() {}
    }
  }
</script>
```

```html
<template>
  <Foo ref="foo" />
</template>

<script>
  export default {
    name: 'Bar',
    mounted() {
      this.$refs.foo.someMethod() // you can access anything on the context
    }
  }
</script>
```

After the change you'll be required to expose your data and methods explicitly.

```html
<template>
  <div />
</template>

<script>
  export default {
    name: 'Foo',
    provide() {
      const { someMethod } = this;
      return {
        someMethod,
      }
    },
    methods: {
      someMethod() {},
      anotherMethod() {},
    }
  }
</script>
```

```html
<template>
  <Foo ref="foo" />
</template>

<script>
  export default {
    name: 'Bar',
    mounted() {
      this.$refs.foo.someMethod() // exposed
      this.$refs.foo.anotherMethod() // error, not exposed
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

Components should not be able to directly access other components context anymore. To declare component's public interface use `provide` and pass any data that should be exposed. The object returned in `provide` would serve as a main access point for the accessor component. To eliminate conflicts and provide better cohesion use `Symbol` for exposed data property names.

A component with such an interface could look like this:

```html
<template>
  <input ref="input">
</template>

<script>
  export const FOCUS_INPUT = Symbol()
  
  export default {
    name: 'MyInput',
    provide() {
      const { focusInput } = this
      return {
        [FOCUS_INPUT]: focusInput
      }
    },
    methods: {
      focusInput() {
        this.$refs.input.focus()
      }
    }
  }
</script>
```

This interface could be utilized as following:

```html
<template>
  <MyInput :ref="myInput" />
</template>

<script>
  import { FOCUS_INPUT } from 'MyInput.vue'
  
  export default {
    mounted() {
      this.$refs.myInput[FOCUS_INPUT]()
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
    provide() {
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
    provide() {
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
    provide() {
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

Example using Composition API:

```html
<template>
  <input ref="inputRef">
</template>

<script>
  import { ref } from 'vue'
  export const INPUT_REF = Symbol()
  
  export default {
    name: 'MyInput',
    setup() {
      const inputRef = ref(null)
      provide(INPUT_REF, inputRef)

      return {
        inputRef
      }
    }
  }
</script>
```

You'll then be able to use it as a ref:

```html
<template>
  <MyInput ref="myInput" />
</template>

<script>
  import { INPUT_REF } from 'MyInput.vue'
  
  export default {
    setup() {
      const myInput = ref(null)
      onMounted(() => {
        console.log(myInput.value[INPUT_REF].value)
      })

      return {
        myInput
      }
    }
  }
</script>
```

# Drawbacks

* Should be triggered after data init, `refs` then would require a lot of fiddling around to preserve reactivity (wrapping exposed values inside an object at a minimum)
* Provides value down the render tree as a side-effect, which may not be the desired behaviour

# Alternatives

* Use dedicated hook instead of `provide` but with exactly the same API. Could be called `expose`.

  **Pros**:

  * Does not cause conflicts with other provisions
  * Clear separation of concerns

  **Cons**:

  * One more option to learn about
  * Could be confusing to pick one between the `provide` and `expose`
  * Does not fix the issue with `refs`

* Use events to expose your component's public interface:  
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