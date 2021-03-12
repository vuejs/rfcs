- Start Date: 2021-03-12
- Target Major Version: 3.x
- Reference Issues: None
- Implementation PR: None

# Summary

Make `attrs` to behave as `props` to allow a similar interface as `React`

# Basic example

```ts
import { defineComponent } from 'vue'

interface CompProps {
  name: string
  age: number
}

// VCA
defineComponent<CompProps>(
  noProps: true,
  setup(props) {
    props.name // string
    props.age // number
  }
)

// VOA
defineComponent<CompProps>({
  noProps: true,
  mounted(){
    this.name // string
    this.age // number
  }
})


// No typescript
// in this case `vm` will have a `Record<string, any>` to allow usage of any prop
defineComponent({
  noProps: true,
  mounted(){
    this.name // no typing
  }
})
```

# Motivation

The main motivation is lowering the bar on the people coming from `React` sometimes, the props runtime checks are not needed, because the developer will rely on the typescript typeschecking.

# Detailed design

In `vue` `props` and `attrs` are different, to be able to achieve this design, `attrs` will behave like props.
When `noProps:true` the component:

- Will have implicitly `inheritAttrs:false`.
- `$attrs` will be bound to the `vm` (just like props are)
- `class` and `style` will not be bound to the `vm` since they will be treated as native and allowed to fallthrough.
- `props` cannot be declared with using `noProps:true`
- If a attribute has the same name as a local property (`computed`, `data`, `methods`, etc), it should warn and do not update or override the local property, that attribute should be added on the `$attrs` or `$props` object

# Drawbacks

- Having an extra `prop` on the options might be a bit confusing
- Having the implicit `inheritAttrs:false` might cause some confusing to more experienced users
- Having `$attrs` bound to the `vm` might cause some unwanted behaviour.
- Increased complexity on the `defineComponent` by having one more way to do it.

# Alternatives

```ts
defineComponent({
  props: null,
})
// or
defineComponent({
  props: false,
})
```

# Adoption strategy

This is completely optional.

# Unresolved questions

There might be some unforeseen behaviour by bounding `$attrs` to the `vm`
