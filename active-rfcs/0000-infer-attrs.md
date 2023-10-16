- Start Date: 2023-01-01
- Target Major Version: 3.x
- Reference Issues:
  - https://github.com/vuejs/core/issues/3452
  - https://github.com/vuejs/core/issues/5423
  - https://github.com/vuejs/core/discussions/6528
  - https://github.com/vuejs/core/discussions/7546
  - https://github.com/vuejs/core/discussions/7981
  - https://github.com/vuejs/core/discussions/8264
  - https://github.com/vuejs/core/discussions/8508
  - https://github.com/vuejs/core/discussions/8620
  - https://github.com/vuejs/core/discussions/8902
  - https://github.com/vuejs/core/discussions/9272
- Implementation PR: 
  - https://github.com/vuejs/core/pull/7444
  - https://github.com/vuejs/core/pull/9413
  - Volar/vue-tsc

# Summary
Allowing to infer attrs by using `attrs` option of `defineComponent` or `defineCustomElement`. 
And in the `setup-script`,  passing generic type in the `defineAttrs<T>` will also infer `attrs` to `T`.

# Basic example

## Using `defineComponent`
Options Api
```tsx
const Comp = defineComponent({
  props: {
    foo: String
  },
  attrs: Object as AttrsType<{
    bar?: number
  }>,
  created() {
    this.$attrs.bar // number | undefined
  }
});

<Comp foo={'str'} bar={1} />;
```

Composition Api
```tsx
const Comp = defineComponent({
  props: {
    foo: {
      type: String,
      required: true
    }
  },
  attrs: Object as AttrsType<{
    bar?: number
  }>,
  setup(props, { attrs }) {
    props.foo; // string
    attrs.bar; // number | undefined
  }
});
<Comp foo={'str'} bar={1} />;
```
Functional Components
```tsx
const Comp = defineComponent(
  (props: { foo: string }, ctx) => {
    ctx.attrs.bar; // number | undefined
    return () => (
      <div>{props.foo}</div>
    )
  },
  {
    attrs: Object as AttrsType<{
      bar?: number
    }>
  }
);
<Comp foo={'str'} bar={1} />;
```


## Using `defineAttrs<T>` in `setup-script`

```vue
// MyImg.vue
<script setup lang="ts">
const attrs = defineAttrs<{ bar?: number }>();
</script>
```
<details open>
<summary>Complie Output: </summary>

```tsx
import { useAttrs as _useAttrs, defineComponent as _defineComponent } from 'vue'

export default /*#__PURE__*/_defineComponent({
  setup(__props, { expose: __expose }) {
  __expose();

      const attrs = _useAttrs()
      
return { attrs }
}

})
```

</details>


## Using `defineCustomElement`
```tsx
const Comp = defineComponent({
  props: {
    foo: String
  },
  attrs: Object as AttrsType<{
    bar?: number
  }>,
  created() {
    this.$attrs.bar // number | undefined
  }
});
```

# Motivation
This proposal is mainly to infer `attrs` using `defineComponent`.

When using typescript in Vue3, the fallthrough attributes is unable to be used. It's not appropriate obviously that only one can be chosen from `typescript` and `Fallthrough Attributes`. In most cases, we choose `typescript` and set attributes to `props` option instead of using the fallthrough attributes.

## Main scenes:

### 1. Wrapping a native HTML element in a new component, such as `img`. 

- Option Api
```tsx
import { defineComponent, type ImgHTMLAttributes, type AttrsType } from 'vue';

const MyImg = defineComponent({
    props: {
        foo: String
    },
    attrs: Object as AttrsType<ImgHTMLAttributes>,
    created() {
        this.$attrs.class // any
        this.$attrs.style // StyleValue | undefined
        this.$attrs.onError // ((payload: Event) => void) | undefined
        this.$attrs.src // string | undefined
    },
    render() {
        return <img {...this.$attrs} />
    }
});

<MyImg class={'str'} style={'str'} src={'https://xxx.com/aaa.jpg'} onError={(e) => {
  e; // Event
}}/>;
```
- Setup Script
```vue
// MyImg.vue
<script setup lang="ts">
import { type ImgHTMLAttributes } from 'vue';
const attrs = defineAttrs<ImgHTMLAttributes>();
</script>
```

### 2. Wrapping a component from UI library in a new component, such as `el-button` from `element-plus`. 

- Option Api
```tsx
import { defineComponent, type AttrsType } from 'vue';

const Comp = defineComponent({
    props: {
        foo: String
    },
    emits: {
        baz: (val: number) => true
    },
    render() {
        return <div>{this.foo}</div>
    }
});

const MyComp = defineComponent({
    props: {
        bar: Number
    },
    attrs: Object as AttrsType<typeof Child>,
    created() {
        this.$attrs.class // unknown
        this.$attrs.style // unknown
        this.$attrs.onBaz; // ((val: number) => any) | undefined
        this.$attrs.foo; // string | undefined
    },
    render() {
        return <Comp {...this.$attrs} />
    }
});

<MyComp class={'str'} style={'str'} bar={1} foo={'str'} onBaz={(val) => { 
    val; // number
}} />;
```

- Setup Script
```vue
// MyButton.vue
<script setup lang="ts">
import { Button } from 'element-plus';
const attrs = defineAttrs<typeof Button>();
</script>
```
# Unresolved questions
Naming suggestions or improvements on the API are welcome.

