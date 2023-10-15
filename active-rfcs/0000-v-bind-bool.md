- Start Date: 2021-11-02
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Introduce new syntax to simplify adding/removing [boolean attributes](https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#boolean-attributes):

- `.bool` modifier to `v-bind`
- `?attr` shorthand for `v-bind:attr.bool`

# Basic example

Consider `wc-button` a [custom element](https://html.spec.whatwg.org/multipage/custom-elements.html) that exposes a `disabled` attribute¹.

```html
<!-- Not using `.bool` modifier -->
<wc-button :disabled="loading">Refresh</wc-button>

<!-- Using `.bool` modifier -->
<wc-button :disabled.bool="loading">Refresh</wc-button>
```

Resulting HTML when `loading` is `true`:

```html
<!-- Not using `.bool` modifier -->
<wc-button disabled="true">Refresh</wc-button>

<!-- Using `.bool` modifier -->
<wc-button disabled>Refresh</wc-button>
```

Resulting HTML when `loading` is `false`:

```html
<!-- Not using `.bool` modifier -->
<wc-button disabled="false">Refresh</wc-button>

<!-- Using `.bool` modifier -->
<wc-button>Refresh</wc-button>
```

# Motivation

In Vue 2.x, setting [enumerated attributes](https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#keywords-and-enumerated-attributes) to `false` coerces them to `"false"` string; otherwise, the attributed is removed.

```html
<!-- `contenteditable` is enumerated -->
<div :contenteditable="false"/> --> <div contenteditable="false"/>

<!-- `disabled` is non-enumerated -->
<input type="text" :disabled="false"> --> <input type="text">
```

Vue 3.x, though, changed this behavior (see [rfc-0024](https://github.com/vuejs/rfcs/blob/master/active-rfcs/0024-attribute-coercion-behavior.md)): all attributes set to `false` are coerced to `"false"` string; attributes are only removed when set to `null` or `undefined`.

That breaking change can be considered net positive:

- it makes working with ARIA attributes easier;
- it generally doesn't break native elements, because Vue 3.x prefers setting `$attrs` as DOM properties, not HTML attributes ([source code](https://github.com/vuejs/vue-next/blob/master/packages/runtime-dom/src/patchProp.ts)).

However, that change can make the interoperability with some custom elements cumbersome, because `"false"` string makes a boolean attribute truthy, potentially introducing bugs.

In those cases, users must refactor their code, replacing `false` with `null` or `undefined` where needed or resorting to idioms like `predicate || null` in templates.

Introducing a `v-bind` modifier to opt-in to Vue 2.x behavior around [boolean attributes](https://html.spec.whatwg.org/multipage/common-microsyntaxes.html#boolean-attributes) would help solve those problems:

- returning `false` instead of `null` or `undefined` is more intuitive when using boolean logic;
- all logic related to showing/hiding attributes is applied directly in the template, making code easier to follow.

# Detailed design

Vue 3.x already provides two `v-bind` modifiers that affect the way `$attrs` are patched:

- `:foo.prop` (shorthand: `.foo`) forces patching `foo` as a DOM property;
- `:foo.attr` forces patching `foo` as a HTML attribute.

Those modifiers are especially useful for interoperability with custom elements.

This RFC introduces a complimentary `v-bind` modifier, `.bool`, to modify how boolean attributes are patched.

Given the following template:

```html
<my-custom-element :active.bool="condition"/>
```

If `condition` is truthy, runtime should set `""` (empty string) to the `active` HTML attribute:

```html
<my-custom-element active/>
```

Otherwise, if `condition` is falsy, runtime should remove `active` HTML attribute from the element, if present:

```html
<my-custom-element/>
```

## Shorthand syntax

For convenience, the `?` prefix can be provided as a shorthand for the `.bool` modifier. Hence, the previous example could also be written as:

```html
<my-custom-element ?active="condition">
```

This shorthand is inspired by `lit-html`, which uses [`?` prefix](https://lit-html.polymer-project.org/guide/template-reference#binding-types) for showing/hiding boolean attributes in templates.

# Adoption strategy

This RFC is unlikely to introduce real-world breaking changes.

Although this feature can help migrate code between Vue 2.x and 3.x, it's possibly not applicable to automatic migration codemods.

# Unresolved questions

## Legibility

The `?` character is possibly too similar to `:` character.

## Usage in render functions

`.foo` in render functions is equivalent to `:foo.prop` in templates, and `^foo` in render functions is equivalent to `:foo.attr` in templates. Perhaps `?foo` can be introduced in render functions as a mnemonic to `:foo.bool` in templates.

# Footnotes

¹ Vue 3.x, by default, tries to patch DOM properties in most cases, resorting to patching HTML attributes only as a fallback. Most of this RFC is only applicable to cases where the HTML element doesn't expose a DOM property with the same name.
