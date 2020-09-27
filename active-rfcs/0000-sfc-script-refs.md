- Start Date: 2020-09-27
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

`<script refs>` is a compilation step executed before `<script setup>`. When using Composition API's responsive objects, you can omit writing `.value`, thereby reducing code and improving coding experience.

# Basic example

```html
<script refs>
export default {
    setup() {
        ref count = 0
        computed odd = count % 2 == 1

        return {
            count,
            odd,
        }
    },
}
</script>
```

Result:

```html
<script>
import { ref, computed } from 'vue'

export default {
    setup() {
        const count = ref(0)
        const odd = computed(() => count.value % 2 == 1)

        return {
            count,
            odd,
        }
    },
}
</script>
```

# Motivation

Usually when writing code, we expect that `.value` is not needed to access variables, and the access method of `.value` has no additional benefit to users. The characteristics of `.value` also cause the code to look messy when non-ref variables are used with ref objects.

`.value` is a unified feature of ref objects, so we think we can use the compiler to remove this feature while retaining the response function.

# Detailed design

Change the script tag to `<script refs>` to enable `refs` compilation.

`<script refs>` adds `ref`, the word `computed` is used as a declaration variable, the variable will be compiled into `ref()`/`computed()`, and the reference of the variable will increase `.value `Suffix.

## Use with `<script setup>`

The `refs` conversion method is not destructive, so it can be used together with `<script setup>`.

```html
<script setup refs>
export ref foo = 1
export let bar = 2
export computed baz = foo + bar
</script>
```

Result:

```html
<script setup>
import { ref, computed } from 'vue'

export const foo = ref(1)
export let bar = 2
export const baz = computed(() => foo.value + bar)
</script>
```

## Don't transform

If you need to reference ref objects, you can still use Composition API in `<script refs>`.

```html
<script lang="ts" refs>
import { ref, Ref } 'vue'

ref foo = 1
let bar = ref(2)

// baz(foo) // type error
baz(bar) // ok

function baz(val: Ref<number>) { ... }
</script>
```

For attributes, the compiler will not perform conversion by default, so setup() can use the original method to return ref objects.

```html
<script refs>
export default {
    setup() {
        ref foo = 1

        return {
            foo,
        }
    }
}
</script>
```

Result:

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

## TypeScript

The type of `refs` objects can be defined like general variables.

```html
<script lang="ts" refs>
ref foo: number | string = 1
computed bar: string = foo
</script>
```

Result:

```html
<script lang="ts">
import { ref, computed } from 'vue'

const foo = ref<number | string>(1)
const bar = computed<string>(() => foo.value)
</script>
```

## Multi-line computed

In order for Language Service to obtain the correct type, multi-line computed needs to be defined as IIFE.

```html
<script refs>
ref foo = 1
ref bar = 2
computed baz = (() => {
    return foo + bar
})()
console.log(baz)
</script>
```

Result:

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

# Drawbacks

- Compiler implementation can be complicated
- Language Service requires additional support

# Adoption strategy

This is an optional and backward compatible new feature
