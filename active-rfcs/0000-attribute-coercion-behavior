- Start Date: 2020-01-30
- Target Major Version: 3.x
- Reference Issues: [#8731](https://github.com/vuejs/vue/issues/8731) [#8735](https://github.com/vuejs/vue/pull/8735) [#9397](https://github.com/vuejs/vue/issues/9397) [#11053](https://github.com/vuejs/vue/issues/11053)
- Implementation PR:

# Summary

- Drop the internal concept of enumerated attributes and treat those attributes the same as normal non-boolean attributes.
- No longer removes attribute if value is boolean `false`. Instead, it's set as `attr="false"` instead. To remove the attribute, use `null` or `undefined`.

# Motivation

In 2.x, we have the following strategies for coercing `v-bind` values:

- For some attribute/element pairs, Vue is always using the corresponding IDL attribute (property): [like `value` of `<input>`, `<select>`, `<progress>`, etc](https://github.com/vuejs/vue/blob/bad3c326a3f8b8e0d3bcf07917dc0adf97c32351/src/platforms/web/util/attrs.js#L11-L18).

- For “[boolean attributes](https://github.com/vuejs/vue/blob/bad3c326a3f8b8e0d3bcf07917dc0adf97c32351/src/platforms/web/util/attrs.js#L33-L40)” and [xlinks](https://github.com/vuejs/vue/blob/bad3c326a3f8b8e0d3bcf07917dc0adf97c32351/src/platforms/web/util/attrs.js#L44-L46), Vue removes them if they are “falsy” ([`undefined`, `null` or `false`](https://github.com/vuejs/vue/blob/bad3c326a3f8b8e0d3bcf07917dc0adf97c32351/src/platforms/web/util/attrs.js#L52-L54)) and adds them otherwise (see [here](https://github.com/vuejs/vue/blob/bad3c326a3f8b8e0d3bcf07917dc0adf97c32351/src/platforms/web/runtime/modules/attrs.js#L66-L77) and [here](https://github.com/vuejs/vue/blob/bad3c326a3f8b8e0d3bcf07917dc0adf97c32351/src/platforms/web/runtime/modules/attrs.js#L81-L85)).

- For “[enumerated attributes](https://github.com/vuejs/vue/blob/bad3c326a3f8b8e0d3bcf07917dc0adf97c32351/src/platforms/web/util/attrs.js#L20)” (currently `contenteditable`, `draggable` and `spellcheck`), Vue tries to [coerce](https://github.com/vuejs/vue/blob/bad3c326a3f8b8e0d3bcf07917dc0adf97c32351/src/platforms/web/util/attrs.js#L24-L31) them to string (with special treatment for `contenteditable` for now, to fix [vuejs/vue#9397](https://github.com/vuejs/vue/issues/9397)).

- For other attributes, we remove “falsy” values (`undefined`, `null`, or `false`) and set other values as-is (see [here](https://github.com/vuejs/vue/blob/bad3c326a3f8b8e0d3bcf07917dc0adf97c32351/src/platforms/web/runtime/modules/attrs.js#L92-L113)).

In 2.x we modeled the concept of “enumerated attributes” as if they can only accept `'true'` or `'false'`, which is technically flawed. This also make them behave differently from other non-boolean attributes which led to confusion. The following table describes how Vue coerce “enumerated attributes” differently with normal non-boolean attributes:

| Binding expr. | `foo` <sup>normal</sup> | `draggable` <sup>enumerated</sup> |
| - | - | - |
| `:attr="null"` | / | `draggable="false"` |
| `:attr="undefined"` | / | / |
| `:attr="true"` | `foo="true"` | `draggable="true"` |
| `:attr="false"` | / | `draggable="false"` |
| `:attr="0"` | `foo="0"` | `draggable="true"` |
| `attr=""` | `foo=""` | `draggable="true"` |
| `attr="foo"` | `foo="foo"` | `draggable="true"` |
| `attr` | `foo=""` | `draggable="true"` |

We can see from the table above, current implementation coerces `true` to `'true'` but removes the attribute if it's `false`. This also led to inconsistency and required users to manually coerce boolean values to string in very common use cases like `aria-*` attributes like `aria-selected`, `aria-hidden`, etc.

# Detailed design

- We intend to drop this internal concept of “enumerated attributes” and treat them as normal non-boolean HTML attributes.

  This solves the inconsistency between normal non-boolean attributes and “enumerated attributes”. It also made it possible to use value other than `'true'` and `'false'`, or even keywords yet to come, for attributes like `contenteditable`.

- For non-boolean attributes, stop removing them if they are `false` and coerce them to `'false'` instead.

  This solves the inconsistency between `true` and `false` and makes outputing `aria-*` attributes easier.

The following table describes the new behavior:

| Binding expr. | `foo` <sup>normal</sup> | `draggable` <sup>enumerated</sup> |
| - | - | - |
| `:attr="null"` | / | / <sup>†</sup> |
| `:attr="undefined"` | / | / |
| `:attr="true"` | `foo="true"` | `draggable="true"` |
| `:attr="false"` | `foo="false"` <sup>†</sup> | `draggable="false"` |
| `:attr="0"` | `foo="0"` | `draggable="0"` <sup>†</sup> |
| `attr=""` | `foo=""` | `draggable=""` <sup>†</sup> |
| `attr="foo"` | `foo="foo"` | `draggable="foo"` <sup>†</sup> |
| `attr` | `foo=""` | `draggable=""` <sup>†</sup> |

<small>†: changed</small>

Coercion for boolean attributes is kept untouched.

# Drawbacks

This proposal is introducing the following breaking changes:

- For “enumerated attributes”:

  - `null` will remove the attribute instead of producing `'false'`.
  - Numbers and string values other than `'true'` and `'false'` won't be coerced to `'true'` any more. (The old behavior shouldn't have been working anyway.)

- For all non-boolean attributes, `false` values won't be removed any more and will be coerced to `'false'`.

The most major breakage here is that users should stop relying on `false` values to remove attributes, instead they are required to use `null` or `undefined` to do that. As boolean attributes are not affected, this change mostly affects enumerated attributes where `'false'` value and absence of the attribute result in different element state. eg. `aria-checked`. It may also affect CSS selectors like `[foo]`.

# Alternatives

N/A

# Adoption strategy

It's unlikely that a codemod can help in this case. We shall provide detailed information in our migration guide and document the coercion behavior in 3.x docs.

## Enumerated attributes

The absence of an enumerated attribute and `attr="false"` may produce different IDL attribute values (which will reflect the actual state), described as follows:

| Absent enumerated attr | IDL attr & value |
| - | - |
| `contenteditable` | `contentEditable` &rarr; `'inherit'` |
| `draggable` | `draggable` &rarr; `false` |
| `spellcheck` | `spellcheck` &rarr; `true` |

To keep the old behavior work, and as we will be coercing `false` to `'false'`, in 3.x Vue developers need to make `v-bind` expression resolve to `false` or `'false'` for `contenteditable` and `spellcheck`.

In 2.x, invalid values were coerced to `'true'` for enumerated attributes. This was usually unintended and unlikely to be relied upon on a large scale. In 3.x `true` or `'true'` should be explicitly specified.

## Coercing `false` to `'false'` instead of removing the attribute

In 3.x, `null` or `undefined` should be used to explicitly remove an attribute.

## Comparison between 2.x & 3.x behavior

<table>
  <thead>
    <tr>
      <th>Attribute</th>
      <th><code>v-bind</code> value <sup>2.x</sup></th>
      <th><code>v-bind</code> value <sup>3.x</sup></th>
      <th>HTML output</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td rowspan="3">2.x “Enumerated attrs”<br><small>i.e. <code>contenteditable</code>, <code>draggable</code> and <code>spellcheck</code>.</small></td>
      <td><code>undefined</code>, <code>null</code>, <code>false</code></td>
      <td><code>undefined</code>, <code>null</code></td>
      <td><i>removed</i></td>
    </tr>
    <tr>
      <td>
        <code>true</code>, <code>'true'</code>, <code>''</code>, <code>1</code>,
        <code>'foo'</code>
      </td>
      <td><code>true</code>, <code>'true'</code></td>
      <td><code>"true"</code></td>
    </tr>
    <tr>
      <td><code>null</code>, <code>'false'</code></td>
      <td><code>false</code>, <code>'false'</code></td>
      <td><code>"false"</code></td>
    </tr>
    <tr>
      <td>Other non-boolean attrs<br><small>eg. <code>aria-checked</code>, <code>tabindex</code>, <code>alt</code>, etc.</small></td>
      <td><code>undefined</code>, <code>null</code>, <code>false</code></td>
      <td><code>undefined</code>, <code>null</code></td>
      <td><i>removed</i></td>
    </tr>
  </tbody>
</table>

# Unresolved questions

N/A
