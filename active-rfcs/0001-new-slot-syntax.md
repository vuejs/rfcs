- Start Date: 2019-01-14
- Target Major Version: 2.x & 3.x
- Reference Issues: https://github.com/vuejs/vue/issues/7740, https://github.com/vuejs/vue/issues/9180, https://github.com/vuejs/vue/issues/9306
- Implementation PR: (leave this empty)

# Summary

Introducing a new syntax for scoped slots usage:

- New directive `v-slot` that unifies `slot` and `slot-scope` in a single directive syntax.

- Shorthand for `v-slot` that can potentially unify the usage of both scoped and normal slots.

# Basic example

Using `v-slot` to declare the props passed to the scoped slots of `<foo>`:

``` html
<!-- default slot -->
<foo v-slot="{ msg }">
  {{ msg }}
</foo>

<!-- named slot -->
<foo>
  <template v-slot:one="{ msg }">
    {{ msg }}
  </template>
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

### Why a new directive instead of fixing `slot-scope`?

If we can go back in time, I would probably change the semantics of `slot-scope` - but:

1. That would be a breaking change now, and that means we will never be able to ship it in 2.x.

2. Even if we change it in 3.x, changing the semantics of existing syntax can cause a LOT of confusion for future learners that Google into outdated learning materials. We definitely want to avoid that. So, we have to introduce something new to differentiate from `slot-scope`.

3. In 3.x, we plan to unify slot types so it's no longer necessary to differentiate between scoped vs. non-scoped slots (conceptually). A slot may or may not receive props, but they are all just slots. With this conceptual unification, having `slot` and `slot-scope` being two special attributes seems unnecessary, and it would be nice to unify the syntax under a single construct as well.

# Detailed design

### Introducing a new directive: `v-slot`.

- It can be used on `<template>` slot containers to denote slots passed to a component, where the slot name is expressed via the **directive argument**:

  ``` html
  <foo>
    <template v-slot:header>
      <div class="header"></div>
    </template>

    <template v-slot:body>
      <div class="body"></div>
    </template>

    <template v-slot:footer>
      <div class="footer"></div>
    </template>
  </foo>
  ```

  If any slot is a scoped slot which receives props, the received slot props can be declared using the directive's attribute value. The value of `v-slot` works the same way as `slot-scope`, so JavaScript argument destructuring is supported:

  ``` html
  <foo>
    <template v-slot:header="{ msg }">
      <div class="header">
        Message from header slot: {{ msg }}
      </div>
    </template>
  </foo>
  ```

- `v-slot` can be used directly on a component, without an argument, to indicate that the component's default slot is a scoped slot, and that props passed to the default slot should be available as the variable declared in its attribute value:

  ``` html
  <foo v-slot="{ msg }">
    {{ msg }}
  </foo>
  ```

### Comparison: New vs. Old

Let's review whether this proposal achieves our goals outlined above:

- Still provide succinct syntax for most common use cases of scoped slots (single default slot):

  ``` html
  <foo v-slot="{ msg }">{{ msg }}</foo>
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
  <foo v-slot="foo">
    <bar v-slot="bar">
      <baz v-slot="baz">
        {{ foo }} {{ bar }} {{ baz }}
      </baz>
    </bar>
  </foo>
  ```

  Notice that **the scope variable provided by a component is also declared on that component itself**. The new syntax shows a clearer connection between slot variable declaration and the component providing the variable.

### More Usage Comparisons

#### Default slot with text

``` html
<!-- old -->
<foo>
  <template slot-scope="{ msg }">
    {{ msg }}
  </template>
</foo>

<!-- new -->
<foo v-slot="{ msg }">
  {{ msg }}
</foo>
```

#### Default slot with element

``` html
<!-- old -->
<foo>
  <div slot-scope="{ msg }">
    {{ msg }}
  </div>
</foo>

<!-- new -->
<foo v-slot="{ msg }">
  <div>
    {{ msg }}
  </div>
</foo>
```

#### Nested default slots

``` html
<!-- old -->
<foo>
  <bar slot-scope="foo">
    <baz slot-scope="bar">
      <template slot-scope="baz">
        {{ foo }} {{ bar }} {{ baz }}
      </template>
    </baz>
  </bar>
</foo>

<!-- new -->
<foo v-slot="foo">
  <bar v-slot="bar">
    <baz v-slot="baz">
      {{ foo }} {{ bar }} {{ baz }}
    </baz>
  </bar>
</foo>
```

#### Named slots

``` html
<!-- old -->
<foo>
  <template slot="one" slot-scope="{ msg }">
    text slot: {{ msg }}
  </template>

  <div slot="two" slot-scope="{ msg }">
    element slot: {{ msg }}
  </div>
</foo>

<!-- new -->
<foo>
  <template v-slot:one="{ msg }">
    text slot: {{ msg }}
  </template>

  <template v-slot:two="{ msg }">
    <div>
      element slot: {{ msg }}
    </div>
  </template>
</foo>
```

#### Nested & mixed usage of named / default slots

``` html
<!-- old -->
<foo>
  <bar slot="one" slot-scope="one">
    <div slot-scope="bar">
      {{ one }} {{ bar }}
    </div>
  </bar>

  <bar slot="two" slot-scope="two">
    <div slot-scope="bar">
      {{ two }} {{ bar }}
    </div>
  </bar>
</foo>

<!-- new -->
<foo>
  <template v-slot:one="one">
    <bar v-slot="bar">
      <div>{{ one }} {{ bar }}</div>
    </bar>
  </template>

  <template v-slot:two="two">
    <bar v-slot="bar">
      <div>{{ two }} {{ bar }}</div>
    </bar>
  </template>
</foo>
```

# Drawbacks

- Introducing a new syntax introduces churn and makes a lot of learning materials covering this topic in the ecosystem outdated. New users may get confused by discovering a new syntax after going through existing tutorials.

  - We need good documentation updates on scoped slots to help with this.

- The default slot usage `v-slot="{ msg }"` doesn't precisely convey the concept that `msg` is being passed to the slot as a prop.

# Alternatives

- New special attribute `slot-props` ([as a previous version of this draft](https://github.com/vuejs/rfcs/blob/8575e72d5a401db5d7206e127e3a6012491d68ed/active-rfcs/0000-new-scoped-slots-syntax.md))
- Directive (`v-scope`) instead of a special attribute as originally proposed in [#9180](https://github.com/vuejs/vue/issues/9180);
- Alternative shorthand symbol (`&`) as suggested by @rellect [here](https://github.com/vuejs/vue/issues/9306#issuecomment-453946190)

# Adoption strategy

The change is fully backwards compatible, so we can roll it out in a minor release (planned for 2.6).

`slot-scope` is going to be soft-deprecated: it will be marked deprecated in the docs, and we would encourage everyone to use / switch to the new syntax, but we won't bug you with deprecation messages just yet because we know it's not a top priority for everyone to always migrate to the newest stuff.

In 3.0 we do plan to eventually remove `slot-scope`, and only support the new syntax. We will start emitting deprecation messages for `slot-scope` usage in the next 2.x minor release to ease the migration to 3.0.

Since this is a pretty well defined syntax change, we can potentially provide a migration tool that can automatically convert your templates to the new syntax.
