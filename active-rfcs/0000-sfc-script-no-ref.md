- Start Date: 2020-09-24
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

`<script noref>`是在`<script setup>`之前執行的編譯步驟，使用Composition API的響應式物件時可以省略編寫`.value`，從而減少代碼和改善編碼體驗。這個方案有以下特點：

- 已有代碼轉為`<script noref>`的修改是對稱的
- 編譯器轉換不具有破壞性
- 不需要額外Language Service支持

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
```
</details>

# Motivation

通常編寫代碼時，我們預期不需要`.value`來存取變量，`.value`的存取方式對用戶沒有額外好處。`.value`的特性也導致非ref變量與ref物件一起使用時，容易出現代碼看起來較混亂的情況。

`.value`是ref物件的統一特性，因此我們認為可以利用編譯器將這個特性抺除，同時可以保留響應功能。

# Detailed design

將script標籤改更為`<script noref>`以啟用`no-ref`編譯。

在變量宣告行結尾或前一行注釋`@ref`/`@computed`，變量會被編譯為`ref()`/`computed()`，並且變量的引用處會增加`.value`後綴。

## Use with `<script setup>`

`noref`轉換不方式不具有破壞性，因此可以與`<script setup>`同時使用。

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

某些情況下需要使用取得ref物件本身而非`.value`，例如在setup() return的時候。

這種情況可以以小括號`()`包裏`no-ref`物件，編譯器會移除`no-ref`物件外的`()`，並抵削一次轉換操作。

```html
<script noref>
let foo = 1 // @ref
let bar = foo
let baz = (foo)
</script>
```

<details>
<summary>Result</summary>

```html
<script>
import { ref } from 'vue'

const foo = ref(1)
let bar = foo.value
let baz = foo
</script>
```
</details>

對於屬性缺省的情況編譯器不會進行轉換，因此setup()可以使用屬性缺省或`()`兩種方式return ref物件。

```html
<script noref>
export default {
    setup() {
        let foo1 = 1 // @ref
        let foo2 = 2 // @ref

        return {
            foo1,
            foo2: (foo2),
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
        const foo1 = ref(1)
        const foo2 = ref(2)

        return {
            foo1,
            foo2: foo2,
        }
    }
}
</script>
```
</details>

## TypeScript

`no-ref`物件可以像一般變量一樣定義類型

```html
<script lang="ts" noref>
let foo: number | string = 1 // @ref
const bar: string = foo; // @computed
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

為了讓Language Service為`// @computed`獲取正確的類型，多行computed需要定義為IIFE。

```html
<script noref>
let foo = 1 // @ref
let bar = 2 // @ref
// @computed
const baz = (() => {
    return foo + bar
})()
console.log(baz);
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
console.log(baz.value);
</script>
```
</details>

# Drawbacks

編譯器實現可能很複雜

# Adoption strategy

這是可選及向後兼容的新功能
