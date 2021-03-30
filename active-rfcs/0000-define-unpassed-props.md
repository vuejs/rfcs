- Start Date: 2021-03-30
- Target Major Version: 3.1
- Reference Issues: https://github.com/vuejs/vue-next/issues/3288 https://github.com/vuejs/vue/issues/4792
- Implementation PR:

# Summary

`props` object doesn't contain all the described props by default, only the ones passed by the parent.
This causes the `const { notPassedRef } = toRefs(props)` to behave different than `toRef(props, 'notPassedRef')`

# Basic example

```ts
defineComponent({
  props: {
    foo: String,
  },
  setup(props) {
    const { foo } = toRefs(props) // props is `{}` making `toRefs` To return `{}`

    // correct
    const foo = toRef(props, 'foo')
  },
})
```

# Motivation

`toRefs(props)` is the recommended way to convert the `props` object into an object of `Record<string, Ref>`, along side of it's single prop sibling `toRef(props, 'prop')`.

I think the current `props` behaviour is just to accommodate the scenario where we need a way to check the if the user as passed something to the component, but I argue if the `prop` value is `undefined` it should mean the user as not passed a value to the `prop`.

```html
<comp />
// are equivalent
<comp :foo="undefined" />
<script>
  props: {
    foo: String
  }
</script>
```

But this behaviour is not consistent with prop type Boolean, because `Boolean` is implicitly `false`

```ts
defineComponent({
  props: {
    foo: Boolean,
  },
  setup(props) {
    props.foo // will be `true` or `false`
  },
})
```

In some situations we won't to know if the user didn't pass the property for that scenario we can define a default, eg:

```ts
defineComponent({
  props: {
    foo: {
      type: Boolean,
      default: undefined,
    },
  },
  setup(props) {
    props.foo // will be `true` or `false` or `undefined`
    // `undefined` will be if the user has not passed a value to the prop
  },
})
```

# Detailed design

`props` should contain all the `props` defined in the `props option`, this will allow the `toRefs` work as expected or `Object.keys(props)` to also return all the valid `props` available.

# Drawbacks

- You can't distinguish `undefined` from a non passed prop.

In this case you can provide a `Symbol` or `object` to make it unique to the component, [playground](https://sfc.vuejs.org/#eyJBcHAudnVlIjoiPHRlbXBsYXRlPlxuICA8Y29tcD48L2NvbXA+XG4gIDxjb21wIDpmb289J3VuZGVmaW5lZCc+PC9jb21wPlxuICA8Y29tcCA6Zm9vPVwie2E6IDF9XCI+PC9jb21wPlxuPC90ZW1wbGF0ZT5cblxuPHNjcmlwdD5cbiBpbXBvcnQgeyBkZWZpbmVDb21wb25lbnQgfSBmcm9tICd2dWUnXG4gaW1wb3J0IENvbXAgZnJvbSAnLi9Db21wLnZ1ZSdcbiAgXG5leHBvcnQgZGVmYXVsdCBkZWZpbmVDb21wb25lbnQoe1xuICBjb21wb25lbnRzOiB7XG4gICAgQ29tcFxuICB9LFxuICBcbn0pO1xuPC9zY3JpcHQ+IiwiQ29tcC52dWUiOiI8dGVtcGxhdGU+XG5cdDxwPlxuICAgIGZvbzoge3tpc0RlY2xhcmVkfX1cbiAgPC9wPlxuPC90ZW1wbGF0ZT5cbjxzY3JpcHQ+XG5cbmNvbnN0IG4gPSB7fVxuXG5leHBvcnQgZGVmYXVsdCB7XG4gIHByb3BzOiB7XG4gICAgZm9vOiB7XG4gICAgIFx0dHlwZTogT2JqZWN0LFxuXHRcdFx0ZGVmYXVsdCgpe1xuICAgICAgICByZXR1cm4gblxuICAgICAgfVxuICAgIH0sXG4gIH0sXG4gIHNldHVwKHByb3BzKXtcbiAgICBjb25zdCBpc0RlY2xhcmVkID0gcHJvcHMuZm9vICE9PSBuO1xuICAgIFxuICAgIHJldHVybiB7XG4gICAgICBpc0RlY2xhcmVkXG4gICAgfVxuIH1cbn1cbjwvc2NyaXB0PiJ9):

```ts
const UNIQUE_INSTANCE = {}

defineComponent({
  props: {
    foo: {
      type: Object,
      default() {
        return UNIQUE_INSTANCE
      },
    },
  },
  setup(props) {
    const isProvided = props.foo !== UNIQUE_INSTANCE
  },
})
```

# Alternatives

If you define the `default` option explicitly it will work as proposed in this RFC.

```ts
defineComponent({
  props: {
    foo: {
      type: String,
      default: undefined,
    },
  },
  setup(props) {
    const { foo } = toRefs(props) // works
  },
})
```

# Adoption strategy

A few users might rely on this behaviour. I would recommend introducing this in 3.1

# Unresolved questions

Any other use cases I'm missing?
