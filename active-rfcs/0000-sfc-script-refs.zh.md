- Start Date: 2020-09-27
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

`<script refs>`是在`<script setup>`之前執行的編譯步驟，使用Composition API的響應式物件時可以省略編寫`.value`，從而減少代碼和改善編碼體驗。

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

編譯結果:

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

通常編寫代碼時，我們預期不需要`.value`來存取變量，`.value`的存取方式對用戶沒有額外好處。`.value`的特性也導致非ref變量與ref物件一起使用時，容易出現代碼看起來較混亂的情況。

`.value`是ref物件的統一特性，因此我們認為可以利用編譯器將這個特性抺除，同時可以保留響應功能。

# Detailed design

將script標籤改更為`<script refs>`以啟用`refs`編譯。

`<script refs>`增加了`ref`, `computed`關鐽字用作宣告變量，變量會被編譯為`ref()`/`computed()`，並且變量的引用處會增加`.value`後綴。

## Use with `<script setup>`

`refs`轉換不方式不具有破壞性，因此可以與`<script setup>`同時使用。

```html
<script setup refs>
export ref foo = 1
export let bar = 2
export computed baz = foo + bar
</script>
```

編譯結果:

```html
<script setup>
import { ref, computed } from 'vue'

export const foo = ref(1)
export let bar = 2
export const baz = computed(() => foo.value + bar)
</script>
```

## Don't transform

如果需要引用ref物件，仍然可以在`<script refs>`中使用Composition API。

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

對於屬性缺省編譯器不會進行轉換，因此setup()可以使用原本的方式return ref物件。

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

編譯結果:

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

`refs`物件可以像一般變量一樣定義類型。

```html
<script lang="ts" refs>
ref foo: number | string = 1
computed bar: string = foo
</script>
```

編譯結果:

```html
<script lang="ts">
import { ref, computed } from 'vue'

const foo = ref<number | string>(1)
const bar = computed<string>(() => foo.value)
</script>
```

## Multi-line computed

為了讓Language Service獲取正確的類型，多行computed需要定義為IIFE。

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

編譯結果:

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

編譯器實現可能很複雜

Language Service需要額外支持

# Adoption strategy

這是可選及向後兼容的新功能
