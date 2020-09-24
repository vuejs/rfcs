- Start Date: 2020-09-24
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

TODO

# Basic example

```html
<script noref>
import { defineComponent } from 'vue'

export default defineComponent({
    setup() {
        let foo = 1 // @ref
        const bar = 2
        const baz = foo + bar // @computed

        console.log(baz)

        return {
            foo: (foo),
            baz: (baz),
        }
    }
})
</script>
```

# Motivation

TODO

# Detailed design

TODO

## Use with `<script setup>`

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

export let foo = ref(1)
export let bar = 2
export const baz = computed(() => foo.value + bar)
</script>
```
</details>

## Multi-line computed

```html
<script noref>
// @computed
export const baz = (() => {
    let foo = 1
    let bar = 2
    return foo + bar
})()
console.log(baz);
</script>
```

<details>
<summary>Result</summary>

```html
<script>
import { computed } from 'vue'

export const baz = computed(() => {
    let foo = 1
    let bar = 2
    return foo + bar
})
console.log(baz.value);
</script>
```
</details>

## Don't transform

```html
<script noref>
let foo = 1 // @ref
const bar = foo
const baz = (foo)
</script>
```

<details>
<summary>Result</summary>

```html
<script>
import { ref } from 'vue'

let foo = ref(1)
const bar = foo.value
const baz = foo
</script>
```
</details>

# Drawbacks

TODO

# Adoption strategy

TODO
