- Start Date: 2020-11-7
- Target Major Version: 2.x / 3.x
- Reference Issues: N/A
- Implementation PR: N/A

Compiler optimization for `with` statements.

用于 `with` 语句的编译优化

# Summary 概要

This RFC introduces an API and compilation method, so that people can simplify the usage of `ref` (omitting `.value`) by using the `with` statement under the premise of fully conforming to JavaScript syntax and semantics.

本RFC介绍一种API和编译方式，让人们可以在完全符合javascript语法和语义地前提下，利用`with`语句简化`ref`的用法（省略`.value`）。

# Basic example 基本示例

```js
const { withRefs } = Vue
const MyComponent = {
  setup() {
    with(withRefs({
      count: 0
    })) {
      const inc = () => count++
      watch(_refs.count, val => {
        console.log(val)
      })
      return {..._refs, inc}
    }
  }
}
```

<details>
<summary>Compiled output</summary>

```js
const { withRefs } = Vue
const MyComponent = {
  setup() {
    {
      const { _refs, _refs: { count } } = Object.getPrototypeOf(withRefs({
        count: 0
      }))
      const inc = () => count.value++
      watch(_refs.count, val => {
        console.log(val)
      })
      return {..._refs, inc}
    }
  }
}
```

</details>

# Motivation 动机

When we introduce a new API `withrefs()`·` (the code is in the detailed design below), even if the source code in the above basic example is not compiled, it can run perfectly in non strict mode, because it fully follows the syntax and semantics of existing JS.

当我们引入一个新的API `withRefs()` 之后（代码在下面的详细设计中），以上基本示例中的源码即使不编译，也是可以在非严格模式下完美运行的，因为他是完全遵循现有JS的语法和语义的。

The above source code has the following advantages:

上述源码有如下优点：

1. For variables of type `ref`, it is no longer necessary to access the bound value in the way of `.value`.

    对于 `ref` 类型的变量，无需再用 `.value` 的方式来访问其绑定的值。
2. It is easy for users to get used to initializing the `ref` value in the literal format of an object.

    用对象字面量的格式来初始化 `ref` 值，用户很容易习惯接受。

Unfortunately, in projects that need to be built, the code is usually compiled in strict mode, which means that we cannot use the `with` statement in these projects.

不幸的是，在需要构建的项目中，代码通常是按照严格模式进行编译的，这就意味着我们无法在这些项目中使用 `with` 语句。

In addition, some drawbacks of the `with` statement are the reasons why it is forbidden in strict mode, and we need to avoid it.

另外， `with` 语句本身存在的一些弊端，是导致它在严格模式中被禁用的原因，也是我们需要避免的。

However, we can use the compiler to allow the use of the `with` statement under certain rules, and optimize the compilation results.

但是我们可以利用编译器，允许在遵循一定的规则的情况下使用 `with` 语句，并对编译结果进行优化。

# Detailed design 详细设计

## New API `withRefs()`: It is designed to be used with the `with` statement to replace the classic `ref` initialization statement.
## 专为配合 `with` 语句设计使用，用于替代经典的 `ref` 初始化语句。

```js
import { reactive, toRefs } from 'vue'

function withRefs(obj) {
  const reactiveObj = reactive(obj)
  const _refs = toRefs(reactiveObj)
  const withObj = Object.create({ _refs })
  Object.keys(_refs).forEach(key => {
    Object.defineProperty(withObj, key, {
      get: () => reactiveObj[key],
      set: val => reactiveObj[key] = val,
    })
  })
  return withObj
}
```

## Specific rules for compiling and optimizing `with` statements
## `with` 语句编译优化的具体规则

1. The `with` expression must be `withRefs()` call, otherwise a compilation error will occur.

    `with` 表达式中必须为 `withRefs()` 调用，否则产生编译错误。

```js
with(withRefs(obj)) {} // ✔
with(obj) {} // ❌ 'with' in strict mode
```

2. Each `with` statement is compiled to generate a `{}` statement block and a `_ref` constant.

    每个 `with` 语句编译生成一个 `{}` 语句块，并生成一个 `_ref` 常量。

3. For the parameter of `withrefs()`, if it is an object literal, then each key name that conforms to the variable name rule will start from `_refs` to deconstruct a constant value. All references to this constant name within the scope of the code block are suffixed with `.value`.

    对于 `withRefs()` 的参数，如果是对象字面量，那么每一个符合变量名规则的键名，都从 `_refs` 上解构出一个常量值，代码块范围内原有对此常量名的引用，全部加上 `.value` 后缀。

4. The `with` statement can be nested, and the compilation rules are consistent with the above. Nested `with` is needed when some initialization values of `Ref` depend on others.

    `with` 语句可以嵌套，编译规则与上面一致。当某些 `Ref` 的初始化值依赖另一些 `Ref` 时，需要用到嵌套的 `with`。

```js
setup() {
  with(withRefs({
    count: 0,
    'weird-key': 'string key will be ignored, and the dash is not a legal identifier charactor',
    100: 'number is not a legal identifier'
  })) {
    const topRefs = _refs // 嵌套的 `with` 内， `_refs` 的值会被覆盖，这里保留一个引用
    console.log(count, result)
    with(withRefs({
      double: computed(() => count * 2),
      triple: computed(() => count * 3),
    })) {
      function inc() {
        count++
      }
      watch(topRefs.count, c => {
        console.log('count:', c)
      })
      watchEffect(() => {
        console.log('double & triple:', double, triple)
      })
      return { ...topRefs, ..._refs, inc }
    }
  }
}
```

<details>
<summary>Compiled Output:</summary>

```js
setup() {
  {
    const { _refs, _refs: { count } } = Object.getPrototypeOf(withRefs({
      count: 0,
      'weird-key': 'string key will be ignored, and the dash is not a legal identifier charactor',
      100: 'number is not a legal identifier',
    })))
    const topRefs = _refs
    console.log(_refs)
    {
      const { _refs, _refs: { double, triple } } = Object.getPrototypeOf(withRefs({
        double: computed(() => count.value * 2),
        triple: computed(() => count.value * 3),
      }))
      function inc() {
        count.value++
      }
      watch(topRefs.count, c => {
        console.log('count:', c)
      })
      watchEffect(() => {
        console.log('double & triple:', double.value, triple.value)
      })
      return { ...topRefs, ..._refs, inc }
    }
  }
}
```

</details>

# Drawbacks 弊端

We have to admit that the writing of the `with` sentence has the following disadvantages:

不得不承认， `with` 语句的写法有以下缺点：

- The method of writing is tedious, which is different from the classical statement

    写法繁琐，与经典的声明语句有些差异

- Sometimes you have to use the nested 'with' to generate unnecessary blocks of statements

    有时不得不用嵌套的 `with` ，会产生不必要的语句块

- Different logic blocks can be divided into several parallel 'with' blocks, but it is not convenient to process the returned values exposed to the template

    不同逻辑块之间可以分成多个并列的 `with` 语句块，但是处理暴露给模板的返回值不方便

However, these shortcomings are not introduced by the compilation method provided by this RFC. If the user is willing to choose this coding style, it means that he has decided to bear these "shortcomings" himself.

但是这些缺点并不是本 RFC 提供的编译方式引入的，如果用户愿意选择这种编码风格的话，那么意味着他是决定自己承担这些“缺点”的。

In addition, due to the implementation of `withRefs()`, the runtime efficiency may be slightly affected.

另外由于 `withRefs()` 的实现方式，运行时效率可能会受到很小的负面影响。

# Alternatives 备选方案

1. For the case where the `withRefs()` parameter is the literal value of the object, the compiled results can be further optimized, and the runtime overhead may be reduced slightly.

    对于 `withRefs()` 参数为对象字面量的情况，编译后的结果可以进一步优化，也许能够稍微减少运行时的开销。

<details>
<summary>Compiled Output:</summary>

```js
setup() {
  {
    const _refs = {
      count: ref(0),
      'weird-key': ref('string key will be ignored, and the dash is not a legal identifier charactor'),
      100: ref('number is not a legal identifier'),
    }
    const { count } = _refs
    const topRefs = _refs
    console.log(_refs)
    {
      const _refs = {
        double: ref(computed(() => count.value * 2)),
        triple: ref(computed(() => count.value * 3)),
      }
      const { double, triple } = _refs
      function inc() {
        count.value++
      }
      watch(topRefs.count, c => {
        console.log('count:', c)
      })
      watchEffect(() => {
        console.log('double & triple:', double.value, triple.value)
      })
      return { ...topRefs, ..._refs, inc }
    }
  }
}
```

</details>

2. We can consider extending the syntax of `with` to further simplify the writing of source code. However, it is not compatible with browser environment. We hope that TC39 can solve this problem in the future, rather than simply disable `with` in strict mode.

    可以考虑过通过扩展 `with` 的语法来进一步简化源码的写法，缺点是无法兼容浏览器环境。希望TC39能够在将来解决这个问题，而不是单纯的在严格模式中禁用 `with`。

# Adoption strategy 采用策略

Fully backwards compatible.

# Unresolved questions 未解决的问题

Dose the identifier used to mount the original reference `_refs` have a more appropriate way of exposure and a more appropriate name?

用于挂载原始引用的标识符 `_refs` 是否有更合适的暴露方式，以及更合适的名称？