- Start Date: 2026-07-20
- Target Major Version: 4.x
- Reference Issues: N/A
- Implementation PR:

> [!NOTE]
> I am not a member of the Vue team, nor am I an expert in Vue's compiler,
> ESTree, or the parser implementations discussed in this RFC. I have tested
> the proposal through a proof of concept and researched the relevant APIs and
> ecosystem usage, but I may still have missed important constraints or made
> incorrect assumptions. Feedback and corrections from Vue maintainers, parser
> authors, and ecosystem maintainers would be greatly appreciated.

# Summary

Standardize the JavaScript and TypeScript AST nodes exposed by
`@vue/compiler-core` and `@vue/compiler-sfc` on an ESTree-compatible contract
owned by Vue.

This proposal changes the public shape of JavaScript expressions embedded in
template ASTs and of script ASTs exposed by the SFC compiler. It does **not**
change Vue's template AST or generated render code.

Vue 4 will upgrade its default parser to `@babel/parser` 8 and enable Babel's
`estree` plugin, including its standard class-feature representation, for every
compiler parse. Babel 8 is required because its TypeScript AST is substantially
closer to typescript-eslint's ESTree dialect than Babel 7's output. A small
normalization layer will remove the remaining parser-specific metadata before
nodes cross a public boundary.

This migration is a prerequisite for parser choice, not the parser-choice API
itself. Once Vue 4 and its ecosystem exchange one parser-neutral AST shape,
later Vue 4 releases can add experimental adapters for compatible parsers such
as Oxc or Yuku without requiring a second ecosystem-wide AST migration. That
would also let integrations such as `@vitejs/plugin-vue` early adopt native-parser
paths against the same contract. If those integrations prove reliable, a later
major could make `@babel/parser` an optional peer used by the Babel adapter and
consider a native parser such as Oxc as the default. This RFC does not commit
Vue 5 to either outcome.

# Basic example

Today, consumers of Vue's exposed script AST use Babel node names and fields:

```ts
if (node.type === 'ObjectProperty' && node.value.type === 'StringLiteral') {
  // ...
}

callExpression.typeParameters
```

Under the proposed contract, the same code uses the Vue 4 AST shape:

```ts
if (
  node.type === 'Property' &&
  node.value.type === 'Literal' &&
  typeof node.value.value === 'string'
) {
  // ...
}

callExpression.typeArguments
```

Some migrations are structural rather than simple renames. For example, a
Babel `ObjectMethod` becomes an ESTree `Property` whose `value` is a function,
and optional chains are represented by a `ChainExpression` wrapper. Consumers
that pass Vue's AST directly to `@babel/traverse` or `@babel/generator` must
change architecture or parse the original source with Babel themselves.

# Motivation

## The current public boundary is parser-owned

Vue uses Babel internally, but Babel's AST is also observable through exported
APIs:

- `SFCScriptBlock.scriptAst` and `scriptSetupAst` contain
  `@babel/types.Statement[]`.
- `SimpleExpressionNode.ast` exposes the parsed AST of a template expression.
- `walkIdentifiers`, `extractIdentifiers`, `isStaticProperty`, and
  `isInDestructureAssignment` accept Babel-shaped nodes.
- `rewriteDefaultAST`, `resolveTypeElements`, and `inferRuntimeType` expose
  Babel node types in their signatures.
- `babelParse` returns Babel's parser result.

Not every one of these exports is documented as a stable end-user API. Some are
labelled internal in source comments. They are nevertheless exported and used
outside Vue, so a major-version migration must treat them as an ecosystem
surface rather than assume that changing them is free.

This coupling means that a parser upgrade can alter Vue's observable AST even
when Vue's intended API has not changed. Babel 8 illustrates the problem: its
TypeScript AST alignment changes fields and node structures from Babel 7,
including `typeParameters` becoming `typeArguments` in several places. Vue
should decide and version the shape it exposes instead of inheriting that
decision implicitly from a dependency.

## Why an ESTree-compatible contract

ESTree is the established common vocabulary for JavaScript ASTs. TypeScript is
different: there is no ECMAScript-standard TypeScript AST. The closest widely
used specification is the extension maintained by typescript-eslint, and real
parsers document deviations from it.

The proposal therefore deliberately says **ESTree-compatible**, not "the
standard TS-ESTree AST". Vue would publish an exact contract:

- JavaScript nodes follow ESTree.
- TypeScript nodes follow the typescript-eslint AST specification where
  applicable.
- Vue documents every intentional deviation, including its position and comment
  model.
- Parser-specific bookkeeping is not part of the public API.

This gives consumers a more familiar and parser-independent vocabulary for
shared JavaScript nodes. It does not guarantee that every ESTree walker,
generator, scope analyzer, or linter understands every TypeScript node. It also
does not make a Babel-specific transform automatically compatible.

Vue already depends on and uses `estree-walker` in both compiler-core and
compiler-sfc. ESTree is therefore not a new abstraction being introduced only
for possible future parsers; part of Vue's compiler already relies on its node
model for traversal. Standardizing the exposed nodes makes that existing
direction explicit and removes the current mixture of an ESTree walker with
Babel-specific public node shapes.

## Why this needs to happen in Vue 4

Changing the exposed AST is a breaking change, so Vue 4 is the practical window
for establishing the contract. Deferring it would leave every later parser
experiment with two poor choices: translate the new parser back into Babel's
proprietary AST, or ask ecosystem tools to support multiple public shapes.

Landing the AST contract in the initial Vue 4 release avoids that second
migration. Experimental Oxc or Yuku integration can then be evaluated in a
later Vue 4 release behind an explicit opt-in. The parser, its packaging, and
its diagnostics can change while the AST consumed by Vue and ecosystem tools
stays stable.

The intended sequence is therefore:

1. Vue 4 adopts Babel 8 with its `estree` output as the public AST.
2. Vue and ecosystem maintainers migrate once to that contract.
3. Later Vue 4 releases may add experimental conforming parser adapters.
4. Only after real ecosystem adoption should Vue consider making Babel optional
   or changing the default parser in a future major.

## Goals

- Make the exposed AST a contract owned and versioned by Vue.
- Prevent a parser dependency upgrade from silently redefining that contract.
- Use established ESTree and typescript-eslint shapes where practical.
- Upgrade to Babel 8 and enable its `estree` plugin consistently across
  compiler-core and compiler-sfc.
- Provide a documented migration path for existing AST consumers.
- Preserve generated client and SSR output for valid inputs.
- Establish the prerequisite for experimental native parser integrations after
  the initial Vue 4 release.

## Non-goals

- Changing Vue's template AST.
- Shipping Oxc, Yuku, or another alternative parser in the initial Vue 4
  release.
- Making `@babel/parser` optional or changing the default parser in Vue 4.
- Adding a global or per-call parser selection API in this RFC.
- Reducing install size, parse time, or duplicate parsing as part of this change.
- Guaranteeing drop-in compatibility with Babel traversal or generation tools.
- Defining a universal TypeScript AST standard for the wider ecosystem.

# Detailed design

## Scope of the AST migration

The following public or publicly observable surfaces move to the Vue-owned
contract:

- `SFCScriptBlock.scriptAst` and `scriptSetupAst`;
- `SimpleExpressionNode.ast` in `@vue/compiler-core`;
- exported helpers that accept or return JavaScript or TypeScript nodes,
  including `walkIdentifiers`, `extractIdentifiers`, `isStaticProperty`,
  `isInDestructureAssignment`, `rewriteDefaultAST`, `resolveTypeElements`, and
  `inferRuntimeType`.

The compiler's template AST nodes, source maps, generated JavaScript, and SFC
descriptor fields unrelated to script ASTs are unchanged.

## Runtime AST shape

The runtime direction is intentionally narrower than the published TypeScript
type decision. JavaScript nodes use ESTree names and fields. TypeScript nodes use
the common shape shared by Babel 8's `estree` output and native parsers that
target the typescript-eslint dialect. Any remaining deviations must be small,
documented, and normalized before the AST is exposed.

The portable position contract is `start` and `end`, expressed as UTF-16 code
unit offsets so `source.slice(node.start, node.end)` works as it does today.
Whether Vue also guarantees `loc`, `range`, comments, or tokens should be
decided with consumers; those fields differ between otherwise compatible
parsers and are not required by Vue's current source-editing implementation.

This RFC defines the vocabulary and compatibility goal. It does not attempt to
freeze every optional property based on the current experiment. The exact
contract should be documented alongside the selected public types before the
Vue 4 release.

## Published TypeScript types

There is no existing type package that is an exact fit:

- `@types/estree` does not include TypeScript nodes.
- `@typescript-eslint/types` describes the desired TypeScript vocabulary, but
  requires `loc` and `range` on every node and follows its own release cadence.

Two practical implementation choices remain:

1. **Use `@oxc-project/types`.** It is compact, actively maintained, and already
   describes an ESTree/TS-ESTree runtime shape close to the proposed contract.
   The trade-off is that Vue's public declarations become coupled to Oxc's
   package, versioning, optional `parent` field, and its more granular
   identifier types. Using it directly would replace a Babel type dependency
   with an Oxc type dependency.
2. **Publish Vue-owned types.** Vue can maintain or generate a small overlay from
   an upstream ESTree/TS-ESTree definition, remove fields it does not promise,
   and re-export the resulting unions from the compiler packages. This keeps
   Vue's public API parser-neutral and under Vue's versioning, but adds ongoing
   maintenance whenever TypeScript syntax evolves. If the declarations are
   generated from `@oxc-project/types`, the published `.d.ts` files should
   contain the resulting Vue types rather than imports from the Oxc namespace.

The current leaning is the second option because the main purpose of the RFC is
to stop a parser vendor from defining Vue's public API. `@oxc-project/types`
remains a reasonable source for generating or validating that contract, and
using it directly remains a viable lower-maintenance alternative. The final
choice should not be inferred from the proof of concept.

## Default parser and normalization

The initial Vue 4 release will parse with `@babel/parser` 8. Every compiler path
that parses JavaScript or TypeScript will enable the `estree` plugin.

That plugin converts Babel-specific JavaScript nodes such as `ObjectProperty`,
`StringLiteral`, and `ClassMethod` into ESTree shapes. Babel 8 is required
because it substantially improves the TypeScript side of this alignment;
Babel 7's `estree` mode still leaves many Babel-specific TypeScript fields
and structures.

Babel output is normalized before it is stored on a public AST field or passed
to a public callback. The normalization removes remaining parser bookkeeping
and makes optional-field representation consistent with Vue's contract.

The normalizer is part of the compiler implementation, not a public conversion
utility. It is tested against the full contract rather than described as a
fixed-size or purely mechanical pass.

## Babel-specific APIs

`babelParse`, `babelParserPlugins`, and compiler-core's expression parser plugin
options explicitly expose Babel today. Babel remains the parser in the initial
Vue 4 release, so removing them is not required for the AST migration itself.
However, deprecating and eventually removing these Babel-specific escape hatches
fits the longer-term goal.

## Future parser integration

After Vue 4 contract ships, a later Vue 4 minor release may add an experimental,
explicitly opt-in adapter for Oxc, Yuku, or another parser that passes the
conformance suite. Because the adapter returns the same AST, compiler internals
and ecosystem integrations can evaluate it without branching on two public node
vocabularies.

This is not only a theoretical convergence target. [Yuku's parser
package](https://github.com/yuku-toolchain/yuku/tree/main/npm/yuku-parser)
explicitly declares that its JavaScript and TypeScript output is the same AST as
Oxc's output. That claim would still need to be verified against Vue's contract,
but it is concrete evidence that independent native parser implementations can
target one external AST shape.

This also creates a migration path beyond compiler-core. For example,
`@vitejs/plugin-vue` and other SFC tooling could adopt the same native-backed
compiler path instead of remaining structurally tied to Babel nodes. Only after
that path works across the ecosystem should a future RFC consider making
`@babel/parser` an optional peer for the Babel adapter or selecting a native
default in Vue 5. Parser selection, configuration, diagnostics, and packaging
remain deliberately open for that later work.

# Validation and ecosystem feedback

The compiler-side change is relatively straightforward. The harder part is
integration across tools that currently consume Babel-shaped nodes.

Feedback is requested from maintainers and users of those APIs to verify the
proposed runtime shape, the published type choice, and which optional metadata
is actually needed. It may be wise to publish the change in a Vue 4 alpha first,
migrate or test several popular integrations against it, and use the results to
adjust the contract and migration guide before a stable release.

# Ecosystem impact

## Confirmed consumer patterns

A limited source survey confirms that this is a real breaking surface:

- `@vitejs/plugin-vue` types `scriptAst` and `scriptSetupAst` as Babel statements
  and compares them structurally for HMR. Its likely migration is primarily a
  type import change, but it still needs an integration test.
- `unplugin-generate-component-name` wraps Vue's statement arrays in a Babel
  `Program` and passes them directly to `@babel/traverse`. It is a hard runtime
  incompatibility, not a node-name-only migration.
- `walkIdentifiers` is used directly by projects including Pruvious,
  VueUse Playground, Pinceau, es-js, and vue-sfc2esm, as well as copied REPL
  compiler code.

## Migration categories

Consumers that only inspect node types and fields can usually migrate to the
new contract directly. A codemod can cover common guard and field changes, but
it cannot safely rewrite every structural transformation.

Consumers that pass nodes to Babel traversal, scope, or generation APIs have
three options:

1. move that logic to tools that support the contract they need;
2. reparse the original source with Babel and keep the transform Babel-native;
3. add an explicit conversion layer in their own package, accepting its cost and
   limitations.

Consumers that parse source independently are unaffected unless they also mix
their AST with nodes returned by Vue.

# Drawbacks

- This is an ecosystem-wide breaking change for an API that is small but used by
  real tools.
- Vue takes long-term ownership of a large JavaScript and TypeScript type
  contract and its normalization rules.
- The migration reduces compatibility with Babel-native traversal and generation
  workflows.
- No longer guaranteeing Babel-only metadata such as attached comments or `loc`
  may affect consumers that rely on currently observable but undocumented
  fields.
- TypeScript interoperability remains dialect-specific; generic ESTree tools might
  not automatically understand TypeScript nodes.
- The initial migration has no runtime performance or dependency-size benefit:
  Babel 8 remains the parser. Its value is enabling later native-parser adoption
  without another public AST migration. Note: there is a chance that this change
  will degrade performance due to `estree` babel plugin.
- The change is effectively irreversible after ecosystem consumers migrate.
- A codemod can assist common cases but cannot make the migration fully
  mechanical.

# Alternatives

## Keep the Babel AST contract

Vue can explicitly document Babel's AST as the supported shape. This has the
lowest migration cost and preserves Babel tooling interoperability, but parser
upgrades that change the AST must then be treated as Vue API changes. More
importantly, it blocks a straightforward future migration to Oxc, Yuku, or
another parser: Vue would either have to reproduce Babel's proprietary AST in an
adapter or impose the same ecosystem-wide shape migration in a later major.

## Normalize a future parser to Babel's shape

Vue could keep its current public contract and adapt another parser to it. This
avoids ecosystem migration but makes Vue responsible for Babel compatibility,
including Babel-only structures and metadata. It would preserve the most
expensive part of the current lock-in and make every native adapter maintain a
Babel emulation layer. It also vastly impacts the performance based on initial
testing.

## Change the default parser at the same time

This combines two independent compatibility changes: public node shape and
parsing behavior. It also makes regressions harder to attribute. A default
parser change should be evaluated separately after the public contract and its
ecosystem value are understood.

## Adopt typescript-eslint's types unchanged

This would reuse a maintained dialect, but its AST contract requires metadata
such as `range` and `loc` that not every potential parser emits. It also does not
by itself define Vue's compatibility policy when that package changes. Vue can
derive its node vocabulary from the specification without delegating versioning
of Vue's public API to it.

# Suggested adoption strategy

A possible rollout would be:

1. Add Vue 3 documentation notices for the AST fields and shape-sensitive
   helpers that will change in Vue 4.
2. Publish a Vue 4 alpha containing Babel 8, the `estree` output, and the
   candidate public types.
3. Migrate or test a representative set of popular consumers, including
   `@vitejs/plugin-vue` and at least one Babel-traverse-based integration.
4. Collect feedback about missing fields, type ergonomics, and migration cases
   that cannot be handled mechanically.
5. Adjust the contract and publish a migration guide and a conservative codemod
   for common node checks.
6. Stabilize the AST with Vue 4 once the alpha has provided enough practical
   evidence.

Existing applications that do not consume compiler ASTs require no source
changes.

After the initial Vue 4 release, experimental native adapters can proceed as
separate features. They must use the same contract and pass the same fixtures;
adding one must not require another migration from ecosystem consumers. Making
Babel optional or changing the default remains a future-major decision based on
real adoption rather than a promise made by this RFC.

# Unresolved questions

1. Is the interoperability and API-ownership benefit large enough to justify a
   major-version break for confirmed Babel-native consumers?
2. Do ecosystem maintainers rely on Babel's currently observable `loc`, attached
   comments, tokens, or other metadata strongly enough that Vue must preserve or
   replace any of them explicitly?
3. Which currently exported but internally described helpers should remain in
   Vue 4 with ESTree signatures, and which should be removed? 
4. Should Vue publish its own generated/maintained AST types, or use
   `@oxc-project/types` directly and accept that dependency as part of the public
   type surface?
