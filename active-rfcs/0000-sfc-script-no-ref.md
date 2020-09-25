- Start Date: 2020-09-24
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

`<script noref>` is a compilation step performed before `<script setup>`. When using Composition API's responsive objects, writing `.value` can be omitted, thereby reducing code and improving coding experience. This program has the following characteristics:

- The modification of the existing code to `<script noref>` is symmetrical
- The compiler conversion is not destructive
- No need for additional Language Service support

# Basic example

```html
<script noref>
export default {
    setup() {
        let count = 0 // @ref
        const inc = () => count++

        return {
            count,
            inc,
        }
    },
}
</script>
```

<details>
<summary>Result</summary>

```html
<script>
import { ref } from 'vue'

export default {
    setup() {
        const count = ref(0)
        const inc = () => count.value++

        return {
            count,
            inc,
        }
    },
}
</script>
```
</details>

# Motivation

Usually when writing code, we expect that `.value` is not needed to access variables, and the access method of `.value` has no additional benefit to users. The characteristics of `.value` also cause the code to look messy when non-ref variables are used with ref objects.

`.value` is a unified feature of ref objects, so we think we can use the compiler to remove this feature while retaining the response function.

# Detailed design

Change the script tag to `<script noref>` to enable `no-ref` compilation.

At the end of the variable declaration line or comment `@ref`/`@computed` in the previous line, the variable will be compiled to `ref()`/`computed()`, and the reference of the variable will be added with a `.value` suffix.

## Use with `<script setup>`

The `noref` conversion method is not destructive, so it can be used together with `<script setup>`.

```html
<script setup noref>
export let foo = 1 // @ref
export let bar = 2
export const baz = foo + bar // @computed
</script>
```

<details>
<summary>Result</summary>

```html
<script setup>
import { ref, computed } from 'vue'

export const foo = ref(1)
export let bar = 2
export const baz = computed(() => foo.value + bar)
</script>
```
</details>

## Don't transform

> To reduce magic, the `()` feature has been removed

If you need to reference ref objects, you can still use the Composition API in the `no-ref` script.

```html
<script lang="ts" noref>
import { ref } 'vue'

let foo = 1 // @ref
let bar = ref(2)

// baz(foo) // type error
baz(bar) // ok

function baz(val: Ref<number>) { ... }
</script>
```

<details>
<summary>Result</summary>

```html
<script lang="ts">
import { ref } 'vue'

let foo = ref(1)
let bar = ref(2)

// baz(foo.value) // type error
baz(bar) // ok

function baz(val: Ref<number>) { ... }
</script>
```
</details>

For attribute less case, the compiler will not perform conversion, so setup() can use the original method to return ref objects.

```html
<script noref>
export default {
    setup() {
        let foo = 1 // @ref

        return {
            foo,
        }
    }
}
</script>
```

<details>
<summary>Result</summary>

```html
<script>
import { ref } from 'vue'

export default {
    setup() {
        const foo = ref(1)

        return {
            foo,
        }
    }
}
</script>
```
</details>

## TypeScript

`no-ref` objects can define types like general variables.

```html
<script lang="ts" noref>
let foo: number | string = 1 // @ref
const bar: string = foo // @computed
</script>
```

<details>
<summary>Result</summary>

```html
<script lang="ts">
import { ref, computed } from 'vue'

const foo = ref<number | string>(1)
const bar = computed<string>(() => foo.value)
</script>
```
</details>


## Multi-line computed

In order for the Language Service to obtain the correct type for `// @computed`, multi-line computed needs to be defined as IIFE.

```html
<script noref>
let foo = 1 // @ref
let bar = 2 // @ref
// @computed
const baz = (() => {
    return foo + bar
})()
console.log(baz)
</script>
```

<details>
<summary>Result</summary>

```html
<script>
import { ref, computed } from 'vue'

const foo = ref(1)
const bar = ref(2)
const baz = computed(() => {
    return foo.value + bar.value
})
console.log(baz.value)
</script>
```
</details>

# Drawbacks

Compiler implementation can be complicated

# Adoption strategy

This is an optional and backward compatible new feature
