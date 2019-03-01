- Start Date: 2019-03-01
- Target Major Version: 2.x
- Reference Issues: https://github.com/vuejs/vue/issues/7582
- Implementation PR: [commit](https://github.com/vuejs/vue/commit/d2902ca8ec5fd184fe81479fea1318553fdb8323) (merged but hidden behind feature flag)

# Summary

Introduce a new shorthand for `v-bind` with `.prop` modifier (binding as DOM properties instead of attributes).

# Basic example

``` html
<!-- current syntax -->
<video v-bind:muted.prop="isMuted"></video>

<!-- with shorthand -->
<video :muted.prop="isMuted"></video>

<!-- with proposed shorthand -->
<video .muted="isMuted"></video>
```

# Motivation

Provide more succinct syntax when binding directly to DOM properties. This is not a very common use case

# Detailed design

This purely a syntax sugar. Any `v-bind` binding with the `.prop` modifier can be shortened to a single `.` followed by the directive argument (the name of the property to bind to).

The behavior is exactly the same with the original, for example it implies a kebab-case to camelCase conversion:

``` html
<div .text-content="someText"></div>
```

This will bind to the `textContent` property of the `<div>`.

`innerHTML` also has a special conversion rule:

``` html
<div .inner-html="someHTML"></div>
```

In string / SFC templates, we can also use uppercase letters in the property name, which makes it more readable:

``` html
<div .innerHTML="someHTML"></div>
```

**Since the syntax is now simpler, we can even consider deprecating `v-html` and `v-text` since they are just sugar for `.innerHTML` and `.textContent`.**

# Drawbacks

- Yet another shorthand. Can be difficult for users to Google for. (But the shorthand maps very well to what it does so it's intuitive enough to be guessed).

- This may look like a modifier at first glance. However a modifier should always follow something to modify, so it shouldn't be difficult to tell them apart.

# Alternatives

N/A

# Adoption strategy

Release notes + updated documentation.
