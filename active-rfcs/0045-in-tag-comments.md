- Start Date: 2026-06-17
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: https://github.com/vuejs/core/pull/14971

# Summary

Allow Vue-specific `//` line comments inside template opening tag attribute
lists. This enables line-level tooling directives, such as `@vue-expect-error`,
to be placed next to the specific attribute or directive they apply to.

These comments are compile-time-only source annotations. They are collected in
the template AST's `comments` property, do not become runtime comments, and do
not alter attribute order or merging semantics.

# Basic example

<!-- prettier-ignore -->
```html
<LegacySelect
  v-model="selectedId"
  :options="options"

  // @vue-expect-error legacy API accepts string IDs at runtime
  :selected-id="selectedId"
/>
```

For generated code, this is equivalent to:

```html
<LegacySelect
  v-model="selectedId"
  :options="options"
  :selected-id="selectedId"
/>
```

# Motivation

The main motivation is precise suppression and expectation of template
diagnostics. Vue's type-checking and language tooling can report errors on
individual attributes and directives inside a multi-line component tag, but
authors currently have no legal comment position that sits next to the specific
attribute or directive that is expected to error. Today, the closest legal
placement is outside the tag:

```html
<!-- @vue-expect-error legacy API accepts string IDs at runtime -->
<LegacySelect
  v-model="selectedId"
  :options="options"
  :selected-id="selectedId"
/>
```

That placement is element-level rather than line-level: it does not clearly bind
to the `:selected-id` attribute, and it becomes less precise as the tag gains
more props, listeners, `v-model` bindings, and ARIA attributes. Placing a
JavaScript comment inside the directive expression only comments the expression
value rather than the surrounding template attribute.

Using `<!-- -->` for this feature would create another problem: Vue already has
normal template comments and a `comments` compiler option that controls whether
they are retained and generated. Giving the same visual syntax different rules
inside opening tags would be confusing. A `//` line comment makes the feature
explicitly Vue-specific and keeps it separate from HTML comments.

The same syntax can document groups of attributes on a large component tag, but
that is a secondary benefit. The primary goal is enabling line-level tooling
directives in multi-line tags.

# Detailed design

An **in-tag line comment** starts with `//` while the Vue template tokenizer is
parsing an opening tag's attribute list. The comment content continues until the
next line terminator. The line terminator is then treated like whitespace and
attribute-list parsing resumes.

In-tag line comments are allowed:

- after the tag name and before the first attribute, once the tag name has been
  separated by whitespace or a line terminator;
- between complete attributes or directives;
- after the last attribute, before the line that contains `>` or `/>`.

For example:

<!-- prettier-ignore -->
```html
<button
  // native button state
  type="button"
  :disabled="pending"

  // accessibility
  aria-live="polite"
>
  Save
</button>
```

Because the syntax is line-based, a tag close on the same line is part of the
comment text. Authors should put the closing `>` or `/>` on a following line:

<!-- prettier-ignore -->
```html
<Comp
  // note
/>
```

## AST representation

In-tag line comments are collected in the template AST's `comments` property.
Each comment entry should record at least:

- the comment kind, so in-tag line comments can be distinguished from other
  source comments if the AST later collects more comment forms;
- the comment content without the leading `//`;
- the source location of the full comment and content.

The exact public type name is an implementation detail, but the AST should allow
tooling to locate an in-tag line comment relative to the following attribute or
directive. For example, Vue language tools can treat an in-tag
`@vue-expect-error` immediately before an attribute or directive as applying to
that following prop entry.

The existing `comments` compiler option does not control in-tag line comments.
That option applies to normal template comments and runtime comment generation.
In-tag line comments are source annotations stored in `ast.comments`; they never
generate runtime comment VNodes.

## Attribute semantics

Comments do not affect the existing order-dependent semantics for attributes.
Existing attribute and directive transforms can continue to apply the same
attribute-order rules.

<!-- prettier-ignore -->
```html
<div
  v-bind="base"
  // explicit class wins according to the existing merge rules
  class="primary"
/>
```

is equivalent to:

```html
<div v-bind="base" class="primary" />
```

## Invalid positions

In-tag line comments are only valid between complete attributes. They are not
valid inside tag names, attribute names, directive names, directive arguments,
modifiers, or attribute values.

The following remain invalid or are parsed according to the existing error
paths:

<!-- prettier-ignore -->
```html
<div cl// no
ass="x" />
<div :[key// no
]="value" />
<div id="a // ordinary attribute text" />
</div // not an opening tag
>
```

Comment-like text inside quoted attribute values remains ordinary attribute
text.

## Template modes

This proposal applies to source strings parsed by Vue's template compiler, such
as SFC templates, inline string templates, and tooling paths that feed source
into `@vue/compiler-dom` or `@vue/compiler-core`.

It does not apply to in-DOM templates because browsers do not preserve this
syntax as Vue source. SFC block opening tags such as `<script setup lang="ts">`
are also out of scope because they are SFC descriptor metadata, not template
content.

## Error handling

Line comments end at a line terminator. If the opening tag itself is left
unterminated after the comment, Vue should report the same diagnostics it
already uses for unterminated tags or malformed attribute lists.

A single `/` that is not followed by another `/` should continue to use the
existing self-closing-tag or malformed-attribute paths. Malformed `<` sequences
inside the attribute list should also keep their existing diagnostics.

## Code generation

In-tag line comments are source-level annotations: parser consumers, language
tools, and formatters can inspect or preserve them, but runtime codegen treats
the element as if the comments were absent. This is what makes line-level
`@vue-expect-error` possible without changing the generated render function.

# Implementation sketch

The compiler tokenizer already has distinct states for scanning tag names,
attribute names, attribute values, and normal comments. The implementation can
recognize `//` from opening-tag attribute-list states and append a comment entry
to `ast.comments`.

At minimum:

- from `BeforeAttrName`, detect `//`, consume through the next line terminator,
  create an in-tag line comment entry, then return to `BeforeAttrName`;
- after a complete attribute or directive has been finalized, allow the same
  `//` handling before the next attribute;
- never enter this state from tag-name scanning, attribute-name scanning,
  `BeforeAttrValue`, attribute value states, directive argument states, or
  closing-tag states;
- do not call the existing `oncomment` callback for in-tag line comments as
  child comments.

The public compiler AST shape changes by adding an `ast.comments` property.

Suggested tests:

- `// @vue-expect-error` before an attribute or directive is preserved in
  `ast.comments` with source location before that prop entry;
- comments before, between, and after attributes, including self-closing tags;
- comments around static attributes, directives, shorthands, dynamic arguments,
  and `v-pre`;
- `comments: true` and `comments: false` do not affect in-tag line comments in
  `ast.comments`;
- unterminated tags, quoted attribute values, and codegen all keep their
  specified behavior.

# Drawbacks

This is Vue-specific syntax and must be documented as compile-time syntax that
is not available in in-DOM templates.

It adds complexity to the HTML tokenizer, especially around attribute states
that already handle malformed and IDE-partial input.

The compiler AST needs a new `comments` collection. Vue tooling and ecosystem
parsers, formatters, syntax highlighters, and language tools will need to
recognize it if they want to support this syntax.

# Alternatives

Keep the current behavior. Users can continue placing comments before the whole
element or relying on blank lines and attribute ordering. This avoids parser and
ecosystem churn, but it does not provide line-level directives for individual
attributes or directives.

Use `<!-- -->` inside opening tags. This matches normal template comments, but
it breaks the expectation that Vue templates remain close to HTML parser syntax
and gives visually identical comments different `comments` option behavior
depending on where they appear.

Treat in-tag annotations as whitespace and discard them during parsing. This
would avoid changing the AST shape, but it would make the feature much less
useful for language tools and formatters that need source locations.

Store in-tag comments inline with element props or children. This would preserve
source order locally, but it would also mix source annotations into runtime
structures. A top-level `ast.comments` collection keeps the annotation
side-channel explicit.

Keep `@vue-expect-error` as an element-level comment before the whole tag. This
avoids syntax inside opening tags, but it loses the line-level precision needed
when only one attribute or directive in a large multi-line tag is expected to
fail type checking.

Use JavaScript comments inside directive expressions:

```html
<Comp :foo="/* important */ foo" />
```

This only works for JavaScript-bearing attributes and does not allow commenting
the surrounding template attribute or directive.

# Adoption strategy

This is an additive template syntax feature. Existing valid templates continue
to compile the same way, and no codemod is required.

Documentation should add a short note to the template syntax guide:

- normal template comments can be preserved or removed according to the
  compiler's `comments` option;
- `//` comments inside opening tag attribute lists are compile-time annotations
  collected in `ast.comments` and never generate runtime comments;
- tooling directives such as `@vue-expect-error` can use in-tag line comments to
  apply to a specific following attribute or directive;
- in-DOM templates do not support this syntax.

Ecosystem projects should be notified before release so that parsers and
formatters can update in the same minor release window where possible.
