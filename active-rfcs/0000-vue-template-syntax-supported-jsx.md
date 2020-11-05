- Start Date: 2020-10-23
- Target Major Version: 2.x / 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

This RFC provides a new method of writing Vue template, which can be mixed with JS code, and supports the syntax of Vue template.  
本RFC提供一种新的Vue模板写法，可以与JS代码混合编写，同时支持Vue Template的语法。

# Basic example

```jsx
const MyComponent = {
  setup() {
    const count = ref(0)
    const inc = () => count.value++
    const max = 10
    return () => (
      <!vt>
        <div> {{ count }} </div>
        <button @click="inc" :disabled="count >= max">inc</button>
      </!vt>
    )
  }
}
```

Equivalent to JSX

```jsx
const MyComponent = {
  setup() {
    const count = ref(0)
    const inc = () => count.value++
    const max = 10
    return () => (
      <>
        <div> { count } </div>
        <button onClick={ inc } disabled={ count >= max }>inc</button>
      </>
    )
  }
}
```

# Motivation

It absorbs the advantages of JSX: it can access any variable according to the rules of JS scope, without manually binding variables to the rendering context.  
It retains the advantages of Vue template: it retains the original syntax and supports the template engine.  
吸取了JSX的优点：能够按照JS作用域的规则访问任何变量，无需手动地将变量绑定到渲染上下文。  
保留了Vue Template的优点：保留原有的语法习惯，同时可以支持模板引擎。

```jsx
const vt = (
  <!vt lang="pug">
  section
    header title
    footer copyright
  </!vt>
)
```

# Detailed design

All JSX codes wrapped in `<!vt></!vt>` are compiled as Vue template format. Different from the classical compilation results, it does not need to rely on the' context 'object, but can access the variables of the JS scope where the code is located.  
所有用`<!vt></!vt>`包裹的JSX代码作为Vue Template格式进行编译处理，与经典的编译结果不同的是，无需依赖`context`对象，但可以访问代码所在的JS作用域的变量。

# Drawbacks

N/A

# Alternatives

N/A

# Adoption strategy

Fully backwards compatible.

# Unresolved questions

N/A