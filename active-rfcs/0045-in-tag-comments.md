- Start Date: 2026-06-17
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR:

# Summary

Allow HTML comments to appear inside Vue opening tag attribute lists in the same
syntactic positions as attributes. This enables line-level tooling directives,
such as `@vue-expect-error`, to be placed next to the specific attribute or
directive they apply to.

These comments are compile-time-only. They are represented in the template AST
alongside attributes and directives, so tooling can inspect and preserve them,
but they do not become runtime comments and do not alter attribute order or
merging semantics.

# Basic example

```html
<LegacySelect
  v-model="selectedId"
  :options="options"

  <!-- @vue-expect-error legacy API accepts string IDs at runtime -->
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

The same syntax can document groups of attributes on a large component tag, but
that is a secondary benefit. The primary goal is enabling line-level tooling
directives in multi-line tags.

The current parser behavior is also surprising. In Vue 3.5, a template like:

```html
<div <!-- note --> id="x"></div>
```

is not treated as a comment. The opening tag ends at the `>` in `-->`, so the
compiler sees attributes named `<!--`, `note`, and `--`, while ` id="x">`
becomes text content. Supporting in-tag comments makes this authoring mistake
parse in the way users usually intended.

# Detailed design

An **in-tag comment** uses the same delimiters as an HTML comment, beginning with
`<!--` and ending with `-->`, and appears while the Vue template tokenizer is
parsing an opening tag's attribute list.

In-tag comments are allowed:

- after the tag name and before the first attribute, with or without whitespace
  after the tag name;
- between complete attributes or directives;
- after the last attribute and before `>` or `/>`.

For example, `<div<!-- note -->>` is valid and is parsed as a `div` element with
an in-tag comment before the tag closes. The comment marks the end of the tag
name and is not part of the tag name.

The comment is represented as a distinct AST node in `ElementNode.props`, not in
`ElementNode.children`. It should record comment content and source location,
and it should be distinguishable from both child comment nodes and prop nodes.
The exact enum name is an implementation detail, but `ElementNode.props` should
be widened from `AttributeNode | DirectiveNode` to include an in-tag comment
node shape, for example `InTagCommentNode`.

This also means the existing `comments` parser option does not affect in-tag
comments. That option controls whether normal child comments are retained as
comment nodes. In-tag comments are always retained in the AST because they are
part of the element's source-level attribute list, but they do not produce
codegen behavior. Transforms that inspect props must explicitly check the node
kind and decide how to handle in-tag comments.

Tooling can interpret the comment content. For example, Vue language tools can
treat an in-tag `@vue-expect-error` immediately before an attribute or directive
as applying to that following prop entry. The compiler's responsibility is to
preserve the node and source order; diagnostic semantics belong to AST
consumers.

## Attribute semantics

Comments do not affect the existing order-dependent semantics for attributes.
Semantic passes that depend on prop order should explicitly distinguish in-tag
comment nodes from attributes and directives, and apply order-dependent
semantics only to attributes and directives. For example:

```html
<div
  v-bind="base"
  <!-- explicit class wins according to the existing merge rules -->
  class="primary"
/>
```

is equivalent to:

```html
<div v-bind="base" class="primary" />
```

Duplicate attribute checks also continue to work across comments, so
`<div id="a" <!-- still duplicate --> id="b" />` emits the same diagnostic as
`<div id="a" id="b" />`.

## Invalid positions

In-tag comments are only valid between complete attributes. They are not valid
inside attribute names, directive names, directive arguments, modifiers, or
attribute values.

The following remain invalid or are parsed according to the existing error
paths:

```html
<div cl<!-- no -->ass="x" />
<div :[key<!-- no -->]="value" />
<div id="a <!-- ordinary attribute text --> b" />
<div id <!-- not between complete attributes --> ="a" />
</div <!-- not an opening tag -->>
```

Comment-like text inside quoted attribute values remains ordinary attribute text.

## Template modes

This proposal applies to source strings parsed by Vue's template compiler, such
as SFC templates, inline string templates, and tooling paths that feed source
into `@vue/compiler-dom` or `@vue/compiler-core`.

It does not apply to in-DOM templates because browsers do not preserve comments
inside an opening tag's attribute list. SFC block opening tags such as
`<script setup lang="ts">` are also out of scope because they are SFC descriptor
metadata, not template content.

## Error handling

Unterminated in-tag comments should emit the same `EOF_IN_COMMENT` diagnostic as
ordinary comments. Nested comments and malformed `<` sequences should follow the
same diagnostics Vue already uses for ordinary comments, attribute names, and
unquoted attribute values.

## AST and code generation

In-tag comments are source-level nodes: parser consumers, language tools, and
formatters can inspect or preserve them, but runtime codegen treats the element
as if the comments were absent. This is what makes line-level
`@vue-expect-error` possible without changing the generated render function.

Transforms that iterate over `ElementNode.props` must explicitly check for
in-tag comment nodes before handling an entry as an `AttributeNode` or
`DirectiveNode`. A transform may use the comment for source-level tooling
semantics, or explicitly do nothing when it only affects runtime behavior.

## Implementation sketch

The compiler tokenizer already has distinct states for scanning attribute names,
attribute values, and normal comments. The implementation can add an in-tag
comment state entered from the attribute-list scanning states when the tokenizer
sees `<!--`.

At minimum:

- from the tag-name scanning state, detect `<!--` after consuming at least one
  tag-name character, finalize the tag name, create an in-tag comment node, then
  return to `BeforeAttrName`;
- from `BeforeAttrName`, detect `<!--`, consume through the next `-->`, create
  an in-tag comment node, then return to `BeforeAttrName`;
- from `AfterAttrName`, allow `<!--` only after finalizing the current attribute
  as a no-value attribute, create an in-tag comment node, then return to
  `BeforeAttrName`;
- never enter this state from `BeforeAttrValue`, attribute value states,
  directive argument states, or closing-tag states;
- do not call the existing `oncomment` callback for in-tag comments as child
  comments. Instead, the parser should append the in-tag comment node to the
  current element's `props` list.

The public compiler AST shape changes by widening `ElementNode.props`.

Suggested tests:

- `@vue-expect-error` before an attribute or directive is preserved in
  `ElementNode.props` before that prop entry;
- comments immediately after the tag name, before/between/after attributes, and
  in self-closing tags;
- comments around static attributes, directives, shorthands, dynamic arguments,
  and `v-pre`;
- `comments: true` and `comments: false` both retain in-tag comments in
  `ElementNode.props`;
- duplicate attributes, unterminated comments, quoted attribute values, and
  codegen all keep their specified behavior;
- transforms that iterate props explicitly distinguish in-tag comments from
  attributes and directives.

# Drawbacks

This further separates Vue template syntax from native HTML and must be
documented as compile-time syntax that is not available in in-DOM templates.

It adds complexity to the HTML tokenizer, especially around attribute states
that already handle malformed and IDE-partial input.

The compiler AST needs a new prop-list node kind. Vue transforms and ecosystem
parsers, formatters, syntax highlighters, and language tools will need to
recognize it instead of assuming every `ElementNode.props` entry is an attribute
or directive.

There is a very small behavior change for templates that currently contain
comment-like text inside opening tags. Such templates are already malformed and
currently produce surprising attributes or text. This proposal changes them to
the behavior authors almost certainly intended.

# Alternatives

Keep the current behavior. Users can continue placing comments before the whole
element or relying on blank lines and attribute ordering. This avoids parser and
ecosystem churn, but leaves long component tags without local documentation.

Allow comments only in SFC templates and not in other string-template compiler
entry points. This would reduce the in-DOM confusion slightly, but it would make
compiler behavior depend on where the same source string came from.

Treat in-tag comments as whitespace and discard them during parsing. This would
avoid changing the AST shape, but it would make the feature much less useful for
language tools and formatters that need to preserve comments in the attribute
list.

Keep `@vue-expect-error` as an element-level comment before the whole tag. This
avoids new syntax inside opening tags, but it loses the line-level precision
needed when only one attribute or directive in a large multi-line tag is
expected to fail type checking.

Introduce a Vue-specific attribute comment syntax such as `v-comment`,
`# comment`, or `// comment`. These alternatives would avoid overloading HTML
comments, but they would create new syntax that is harder to explain and less
consistent with the rest of Vue templates.

Use JavaScript comments inside directive expressions:

```html
<Comp :foo="/* important */ foo" />
```

This only works for JavaScript-bearing attributes and does not allow commenting
groups of attributes.

# Adoption strategy

This is an additive template syntax feature. Existing valid templates continue
to compile the same way, and no codemod is required.

Documentation should add a short note to the template syntax guide:

- comments between child nodes can be preserved or removed according to the
  compiler's `comments` option;
- comments inside opening tag attribute lists are represented in the AST as
  compile-time annotations and never generate runtime comments;
- tooling directives such as `@vue-expect-error` can use in-tag comments to
  apply to a specific following attribute or directive;
- in-DOM templates do not support this syntax.

Ecosystem projects should be notified before release so that parsers and
formatters can update in the same minor release window where possible.

# Unresolved questions

Should Vue accept the same non-canonical comment forms as ordinary template
comments, or should in-tag comments require the canonical `<!-- ... -->`
form only?
