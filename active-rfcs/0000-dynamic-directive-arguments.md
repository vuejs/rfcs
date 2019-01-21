- Start Date: 2019-01-16
- Target Major Version: (2.x / 3.x)
- Reference Issues: https://github.com/vuejs/rfcs/pull/2, https://github.com/vuejs/rfcs/pull/3
- Implementation PR: (leave this empty)

# Summary

Support dynamic values in directive arguments.

# Basic example

``` html
<div v-bind:[key]="value"></div>
<div v-on:[event]="handler"></div>
```

# Motivation

Due to directive arguments being static, currently users would have to resort to argument-less object bindings in order to leverage dynamic keys:

``` html
<div v-bind="{ [key]: value }"></div>
<div v-on="{ [event]: handler }"></div>
```

However, this has a few issues:

- It's a lesser known technique that relies on knowledge of the object-based syntax for `v-bind`/`v-on` and the existence of computed property keys in JavaScript.

- It generates less efficient code: an ad-hoc object is allocated and if there are other static bindings on the same element, it has to be dynamically iterated and mixed into the existing data object. The code looks roughly like this:

  ``` js
  return h('div', {
    on: Object.assign({
      click: onClick
    }, {
      [event]: handler
    })
  })
  ```

  Where as with dynamic arguments we can directly generate:

  ``` js
  return h('div', {
    on: {
      click: onClick,
      [event]: handler
    }
  })
  ```

In addition, `v-slot` doesn't have an equivalent object syntax, since it's value is used for declaring the slot scope variable. So without the dynamic argument, the `v-slot` syntax will not be able to support dynamic slot names. Although this is probably a very rare use case, it would be a pain having to rewrite an entire template into render function just because of this single limitation.

# Detailed design

``` html
<!-- v-bind with dynamic key -->
<div v-bind:[key]="value"></div>

<!-- v-bind shorthand with dynamic key -->
<div :[key]="value"></div>

<!-- v-on with dynamic event -->
<div v-on:[event]="handler"></div>

<!-- v-on shorthand with dynamic event -->
<div @[event]="handler"></div>

<!-- v-slot with dynamic name -->
<foo>
  <template v-slot:[name]>
    Hello
  </template>
</foo>

<!-- v-slot shorthand with dynamic name -->
<!-- pending #3 -->
<foo>
  <template #[name]>
    Default slot
  </template>
</foo>
```

# Drawbacks

- Theoretically this opens up the directive argument to arbitrarily complex JavaScript expressions, but html attribute names cannot contain spaces and quotes, so in some cases the user may get tripped up with something like:

  ``` html
  <div :[key + 'foo']="value"></div>
  ```

  Which does not work as expected. A workaround would be:

  ``` html
  <div :[`key${foo}`]="value"></div>
  ```

  That said, complex dynamic key bindings should probably be pre-transformed in JavaScript via a computed property.

- Allowing dynamic arguments for all directives means custom directive implementations now also need to account for potential argument changes in addition to value changes.

  This also requires the addition of `binding.oldArgs` to the custom directive binding context.

# Alternatives

N/A

# Adoption strategy

This is non-breaking and should be straightforward to introduce with appropriate documentation updates.

# Unresolved questions

What kind of expressions should be allowed
