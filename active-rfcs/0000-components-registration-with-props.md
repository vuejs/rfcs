- Start Date: 2020-3-7
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

Components registration with default props when define component, default props will auto injected. 

# Basic example

Provide default props on component registration.
```javascript
const Panel = {
  components: {
    Card: [Card, { shadow: 'hover' }]
  },
  template: `<Card />` // Is equivalent to: <Card shadow="hover">
}
```

# Motivation

To reduce write same component props, such as
```html
  <Card shadow="hover"/>
  <Card shadow="hover"/>
  <Card shadow="hover"/>
```
This scenario is often repeat but necessary when using a components library.

There are many similar situations, component always has some of the same props on the same view, such as `<Button plain />`, `<Input clearable />`

# Detailed design

Expand PublicAPIComponent signature
```typescript
type PublicAPIComponentWithProps<T extends ComponentOptions> = [T, Props<T>]
```

Provide default props on component registration when define component.
```javascript
const Panel = {
  components: {
    Card: [Card, { shadow: 'hover' }]
  },
  template: [
    `<Card />`, // Is equivalent to: <Card shadow="hover">
    `<Card shadow="always" />`, // 'always' cover 'hover'
    `<Card otherProps />` // <Card shadow="hover" otherProps>
  ].join('')
}
```

When use component without same props, default props will injected, like:
```javascript
const props = Object.assign({}, defaultProps, templateProps)
```

# Drawbacks

- Runtime cost
- Increased bundle size, it was hard to tree shaked
- More complicated typescript signature

# Alternatives

There are a similar signature, but it takes more chars.
```typescript
type PublicAPIComponentWithProps<T extends ComponentOptions> = {
  component: T,
  props: Props<T>
}
```

# Adoption strategy

No migration steps required.

# Unresolved questions

- Should it work with `Ref`?
- Shall we need to deep clone default props?
