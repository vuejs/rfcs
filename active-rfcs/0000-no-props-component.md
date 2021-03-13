- Start Date: 2021-03-12
- Target Major Version: 3.x
- Reference Issues: None
- Implementation PR: None

# Summary

Allow defining props by Typescript interface

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

People coming form other frameworks (ReactJS) are used to pass `Props` interface as a first generic and expecting to "be working" when using `props`. That's not possible because `vue` has runtime checks and the props will be inferred by the `props` object.
Other case could be: No need to have runtime validation.

# Detailed design

In `vue` everything declared in the `props` object will be bound to the component instance, allowing the direct access in the template (without `$props.` prefix), everything else will be added to `$attrs` object.

When passing the first generic to `defineComponent` (eg: `defineComponent<{ title: string }>`), is expected that will be `props` and have the same behaviour, but in reality that is not possible since what drives the assigning to `$props`/`$attrs` is if that `prop` name is defined in `props` object.

To achieve what's proposed in this RFC, attributes will need to be treated as props, for that to happen we need an extra option (eg: `noProps: true`) or passing `props:null`.

If `noProps: true`:

- Will have implicitly `inheritAttrs:false`.
- `$attrs` will be bound to the `vm` (just like props are)
- `class` and `style` will not be bound to the `vm` since they will be treated as native and allowed to fallthrough.
- `props` cannot be declared with using `noProps:true`
- If a attribute has the same name as a local property (`computed`, `data`, `methods`, etc), it should warn and do not update or override the local property, that attribute should be added on the `$attrs` or `$props` object

# Drawbacks

- Having an extra `prop` on the options might be a bit confusing
- Having the implicit `inheritAttrs:false` might cause some confusing to more experienced users
- Having `$attrs` bound to the `vm` might cause some unwanted behaviour.
- Increased complexity on the `defineComponent` by having one more way to do handle the props from generic type.

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
