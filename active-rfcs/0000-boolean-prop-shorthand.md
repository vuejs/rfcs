- Start Date: (2019-09-07)
- Target Major Version: (2.x / 3.x)
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

A shorthand for negating a boolean prop.

# Basic example

```html
<!-- OLD -->
<my-component :standalone="false"></my-component>
<!-- NEW -->
<my-component !standalone></my-component>
```

# Motivation

> (NEED TO DO RESEARCH AND GET THE NUMBERS)

When consuming components, from famous component libraries like Element UI, Vuetify or similar, we see there are a lot of boolean properties in play. Boolean properties occupy a major share on the consumption side as well.
With current vue template syntax, setting a boolean property to true is really easy. By just adding the prop name in the template automatically sets it to true within the consumed component. Setting a property to false however requires the use of `v-bind` syntax or the `:` shorthand and then explicitly setting it to `false`. It isn't as clean as setting the prop to true. Hence my proposal.

It helps by addressing multiple problems.

- Avoid accidentally setting a boolean property to string `"false"` when the v-bind shorthand `:` is missed.
- Readability. Example -

```html
<my-component :standalone="false"></my-component>
```

I read this as _standalone false_ in my head but what I really want my head to read _not standalone_

- Shorter code. `:standalone="false"` seems a lot than its `true` value counterpart.

# Detailed design

The new syntax will allow anyone to set a boolean property to false by just adding an exclamation mark (`!`) before the prop name in template.

```html
<!-- OLD -->
<my-component :standalone="false"></my-component>
<!-- NEW -->
<my-component !standalone></my-component>
```

Users can still use the v-bind syntax to set prop to false or an expression. Having this syntax will allow users to use the shorthand for simpler cases and then use the v-bind syntax when they really need to evaluate an expression.

This also shortens the code by _8_ characters at the least everywhere a boolean property is just set to false.

It also forces users to read the code like _not standalone_ which is a better mental construct a lot of the times than reading _standalone false_

It can also lead to slight optimization on the template compiler side. When the compiler encounters a `!` before a property it can just assume the prop value to be static `false` instead of looking at the RHS, evaluating it and then setting it to false.

# Drawbacks

- May be because it seems like a really small one off case.
- Adding support for this in template might bloat the framework.
- May be this can result in ambiguous parsing rules.

# Alternatives

# Adoption strategy

The proposal doesn't introduce any breaking change. People should be able to use it in their existing projects without any hiccups.

# Unresolved questions

- Did I articulate the problem well?
- I am not sure what other drawbacks I may not be seeing.
- Is it possible to analyze a project like Vuetify and get a real number on what percentage of boolean properties they have?
