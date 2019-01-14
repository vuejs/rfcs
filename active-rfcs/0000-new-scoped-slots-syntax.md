- Start Date: 2019-01-14
- Target Major Version: 2.x & 3.x
- Reference Issues: https://github.com/vuejs/vue/issues/7740, https://github.com/vuejs/vue/issues/9180, https://github.com/vuejs/vue/issues/9306
- Implementation PR: (leave this empty)

# Summary

Introducing a new syntax for scoped slots usage:

- `slot-props` replaces `slot-scope` with slightly different semanticsp;
- Shorthand for `slot-props` that can potentially unify the usage of both scoped and normal slots.

# Basic example

Using `slot-props` to declare the props passed to the default slot of `<foo>`:

``` vue
<foo slot-props="{ msg }">
  {{ msg }}
</foo>
```

Same example, with shorthand (shorthand symbol is tentative):

```
<foo ()="{ msg }">
  {{ msg }}
</foo>
```

# Motivation

When we first introduced scoped slots, it was verbose because it required always using `<template slot-scope>`:

``` html
<foo>
  <template slot-scope="{ msg }">
    <div>{{ msg }}</div>
  </template>
</foo>
```

To make it less verbose, in 2.5 we introduced the ability to use `slot-scope` directly on the slot element:

``` html
<foo>
  <div slot-scope="{ msg }">
    {{ msg }}
  </div>
</foo>
```

This means it works on component as slot as well:

``` html
<foo>
  <bar slot-scope="{ msg }">
    {{ msg }}
  </bar>
</foo>
```

However, the above usage leads to a problem: the placement of `slot-scope` doesn't always clearly reflect which component is actually providing the scope variable. Here `slot-scope` is placed on the `<bar>` component, but it's actually defining a scope variable provided by the default slot of `<foo>`.

This gets worse as the nesting deepens:

``` html
<foo>
  <bar slot-scope="foo">
    <baz slot-scope="bar">
      <div slot-scope="baz">
        {{ foo }} {{ bar }} {{ baz }}
      </div>
    </baz>
  </bar>
</foo>
```

It's not immediately clear which component is providing which variable in this template.

Someone suggested that we should allow using `slot-scope` on a component itself to denote its default slot's scope:

``` html
<foo slot-scope="foo">
  {{ foo }}
</foo>
```

Unfortunately, this cannot work as it would lead to ambiguity with component nesting:

``` html
<parent>
  <foo slot-scope="foo"> <!-- provided by parent or by foo? -->
    {{ foo }}
  </foo>
</parent>
```

This is why I now believe allowing using `slot-scope` without a template was a mistake.

### Why a new attribute instead of fixing `slot-scope`?

If we can go back in time, I would probably change the semantics of `slot-scope` - but:

1. That would be a breaking change now, and that means we will never be able to ship it in 2.x.

2. Even if we change in in 3.x, changing the semantics of existing syntax can cause a LOT of confusion for future learners that Google into outdated learning materials. We definitely want to avoid that. So, we have to introduce a new attribute to differentiate from `slot-scope`.

# Detailed design

### Introducing a new special attribute: `slot-props`.

- It can be used on a component to indicate that the component's default slot is a scoped slot, and that props passed to this slot will be available as the variable declared in its attribute value:

  ``` html
  <foo slot-props="{ msg }">
    {{ msg }}
  </foo>
  ```

- It can also be used on `<template>` slot containers (exactly the same usage as `slot-scope` in this case):

  ``` html
  <foo>
    <template slot="header" slot-props="{ msg }">
      Header msg: {{ msg }}
    </template>

    <template slot="footer" slot-props="{ msg }">
      Footer msg: {{ msg }}
    </template>
  </foo>
  ```

- It **can NOT** be used on normal elements. This means for named slots, **a `<template>` wrapper for that slot is required.**

### Shorthand Syntax

`slot-props` also has a shorthand syntax: `()` (tentative, open to suggestions).

The above examples using shorthand syntax:

``` html
<foo ()="{ msg }">
  {{ msg }}
</foo>

<foo>
  <template slot="header" ()="{ msg }">
    Header msg: {{ msg }}
  </template>

  <template slot="footer" ()="{ msg }">
    Footer msg: {{ msg }}
  </template>
</foo>
```

### Shorthand Syntax for Named Slots

The shorthand syntax can potentially use its attribute name to denote a named slots as well. The above named slots example can thus be written as:

``` html
<foo>
  <template (header)="{ msg }">
    Header msg: {{ msg }}
  </template>

  <template (footer)="{ msg }">
    Footer msg: {{ msg }}
  </template>
</foo>
```

### Shorthand Syntax for Normal Slots

It can even be applied to normal, non-scoped slots, effectively unifying both types of slots in a single syntax:

``` html
<foo>
  <template (header)>
    This is header
  </template>

  <template (footer)>
    This is footer
  </template>
</foo>
```

Considering in v3 we plan to eliminate the conceptual difference between normal vs. scoped slots, this syntax unification seems like a good match.

### Note on shorthand syntax

The tentative shorthand is `()` because it resembles the starting parens of an arrow function and loosely relates to "creating a scope". An arrow function is also typically used for render props, the equivalent of scoped slots in JSX.

Potential drawback: Angular uses `()` in their templates for a different use case. This may confuse users coming from Angular or have to work with both frameworks.

### Comparison: New vs. Old

Let's review whether this proposal achieves our goals outlined above:

- Still provide succinct syntax for most common use cases of scoped slots (single default slot):

  ``` html
  <foo ()="{ msg }">{{ msg }}</foo>
  ```

- Clearer connection between scoped variable and the component that is providing it:

  Let's take another look at the deep-nesting example using current syntax (`slot-scope`) - notice how slot scope variables provided by `<foo>` is declared on `<bar>`, and the variable provided by `<bar>` is declared on `<baz>`...

  ``` html
  <foo>
    <bar slot-scope="foo">
      <baz slot-scope="bar">
        <div slot-scope="baz">
          {{ foo }} {{ bar }} {{ baz }}
        </div>
      </baz>
    </bar>
  </foo>
  ```

  This is the equivalent using the new syntax:

  ``` html
  <foo ()="foo">
    <bar ()="bar">
      <baz ()="baz">
        {{ foo }} {{ bar }} {{ baz }}
      </baz>
    </bar>
  </foo>
  ```

  Notice that **the scope variable provided by a component is also declared on that component itself**. The new syntax shows a clearer connection between slot variable declaration and the component providing the variable.

### More Usage Comparisons

Here are [some more usage examples](https://gist.github.com/yyx990803/c6a7f63219825cb90105aad1a8a17e3b) using both the new and old syntax.

# Drawbacks

- Introducing a new syntax introduces churn and makes a lot of learning materials covering this topic in the ecosystem outdated. New users may get confused by discovering a new syntax after going through existing tutorials.

  - We need good documentation updates on scoped slots to help with this.

- Some may think this aesthetically is "non-HTML", but we already have shorthands like `:` and `@`. Slots is an important component mechanism which I think having its own dedicated shorthand seems justified.

# Alternatives

- Directive (`v-scope`) instead of a special attribute as originally proposed in [#9180](https://github.com/vuejs/vue/issues/9180);
- Alternative shorthand symbol (`&`) as suggested by @rellect [here](https://github.com/vuejs/vue/issues/9306#issuecomment-453946190)

# Adoption strategy

The change is fully backwards compatible, so we can roll it out in a minor release (planned for 2.6).

`slot-scope` is going to be soft-deprecated: it will be marked deprecated in the docs, and we would encourage everyone to use / switch to the new syntax, but we won't bug you with deprecation messages just yet because we know it's not a top priority for everyone to always migrate to the newest stuff.

In 3.0 we do plan to eventually remove `slot-scope`, and only support `slot-props` and its shorthand. We will start emitting deprecation messages for `slot-scope` usage in the next 2.x minor release to ease the migration to 3.0.

Since this is a pretty well defined syntax change, we can potentially provide a migration tool that can automatically convert your templates to the new syntax.

# Unresolved questions

- Settle on a good shorthand symbol
