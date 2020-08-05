- Start Date: 2020-08-05
- Target Major Version: 2.x / 3.x
- Reference Issues: https://github.com/vuejs/vue-next/issues/1779
- Implementation PR: N/A

# Summary

This RFC provides a method to merge data into the component template rendering context in the `setup()` function of the composition API, which can replace the current practice of passing the data to be merged through the return value.  
本RFC提供一种在Composition API的`setup()`函数中将数据合并到组件模板渲染上下文的方法，可替代当前通过返回值传递要合并数据的做法。

# Basic example

```javascript
const MyComponent = {
  setup(props, {merge}) {
    const {foo} = useFoo()
    merge({foo})

    const {bar} = useBar()
    merge({bar})

    const {baz, update} = useBaz()
    function clickHandler() {
      // ...
    }
    merge({baz, update, clickHandler})
  }
}
```

Equivalent to

```javascript
const MyComponent = {
  setup() {
    const {foo} = useFoo()

    const {bar} = useBar()

    const {baz, update} = useBaz()
    function clickHandler() {
      // ...
    }

    return {
      foo,
      bar,
      baz,
      update,
      clickHandler
    }
  }
}
```

# Motivation

Returning all data in the return statement will lead to verbosity statement and cumbersome maintenance.  
在返回语句中返回全部数据会导致语句冗长，维护烦琐。

The logic and the return statement are separated in separate blocks of code, which looks like a remnant of the drawbacks of the Options API.  
逻辑和返回语句被分隔在不同的代码块，看上去就像是 Options API 弊端的残留。

In fact, to merge data into a rendering context, you don't have to use returning values.  
实际上若要在渲染上下文中合并数据并非只能通过返回值。

Using an API function to merge data can make the logically related code no longer be separated by the `return` statement, making the code appear cleaner and easier to maintain.  
通过一个接口函数来合并数据，可以让逻辑相关的代码不再被`return`语句分隔，让代码显得更加整洁、便于维护。

# Detailed design

Add a function property `merge` to the second argument `context` of the `setup()` function. Use this function to merge data into the template rendering context.  
在`setup()`函数的第二个参数`context`上增加一个函数属性`merge`，使用此函数可以把数据合并到模板渲染上下文。

This function can accept an object as an argument, and its internal implementation is consistent with the current processing of the return value of `setup()`.  
此函数可接受一个对象做参数，其内部实现与当前对`setup()`返回值的处理保持一致。

This function can be called multiple times and can coexist with the return value. Duplicate name properties will be overridden in the merge order. The return value will be merged last.  
此函数可以被多次调用，可以与返回值并存。重名属性将按照合并顺序被覆盖。返回值将被最后合并。

```javascript
const dataArrayToBeMerged = []
function merge(data) {
  dataArrayToBemerged.push(data)
}

context.merge = merge

const returnedData = options.setup(props, context)
if (typeof returnedData != 'function') {
  merge(returnedData)
}

const dataToBeMerged = Object.assign({}, ...dataArrayToBeMerged)

// Merge `dataToBeMerged` to the template rendering context 
// ...
```

# Drawbacks

Almost no shortcomings, simple implementation, simple use, does not affect other functions, fully compatible with the existing API.  
几乎没有缺点，实现简单，使用简单，不影响其他功能，完全兼容现有API。

The only inconvenience is that this function may be used quite frequently as an alternative to the return statement, but it is not an independent argument of `setup()`.  
唯一的不便就是，作为返回语句的替代，此函数的使用频率可能会相当高，但它并不是`setup()`的独立参数。

# Alternatives

- In the first alternative, considering the possible frequency of use, I prefer to provide this function as the first independent argument to the `setup()` function. But this is not compatible with the current API.  
  第一种替代方案，考虑到可能的使用频率，我更希望能够将此函数作为排在首位的独立参数提供给 `setup()` 函数。但这会与当前的API不兼容。

- The second alternative is to use this function as the third independent argument of `setup()`. This is compatible with the current API. The disadvantage is that the first two arguments need to be declared when defining `setup()`, even if they are not used.  
  第二种替代方案，将此函数作为`setup()`的第三个独立参数。这能够与当前API兼容。缺点是，定义`setup()`时需要同时声明前两个参数，即使他们不会被用到。

- The third alternative is to add `setup_v2(merge, props, context)`, coexists with the current `setup(props, context)`. This is also compatible with the current API. It just doesn't look so elegant.  
  第三种替代方案，增加一个`setup_v2(merge, props, context)`，与当前的`setup(props, context)`并存。这也与当前API兼容。只是看起来好像不那么优雅。

# Adoption strategy

Fully backwards compatible.

# Unresolved questions

None