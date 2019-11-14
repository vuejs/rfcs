- Start Date: 2019-11-12
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Remove support for [filters](https://vuejs.org/v2/guide/filters.html).

# Basic example

``` html
<!-- before -->
{{ msg | format }}

<!-- after -->
{{ format(msg) }}
```

# Motivation

- Filters' functionality can be easily replicated with method calls or computed properties so it provides primarily syntactical rather than practical value.

- Filters requires a custom micro syntax that breaks the assumption of expressions being "just JavaScript" - which adds to both learning and implementation costs. In fact, it conflicts with JavaScript's own bitwise or operator (`|`) and makes expression parsing more complicated.

- Filters also create extra complexity in template IDE support (again due to them not being really JavaScript).

# Drawbacks

- Filters read a bit nicer when chaining multiple filters vs. calling multiple functions:

  ``` js
  msg | uppercase | reverse | pluralize
  // vs
  pluralize(reverse(uppercase(msg)))
  ```

  However in practice we find chaining with a count beyond two to be fairly rare, so the readability loss seems acceptable.

- Individually importing or defining methods can be a bit more boilerplate-y than globally registered filters. However, global filters are not fundamentally different from registering specially named global helpers on `Vue.prototype`. Such global registration comes with trade-offs: they make the code dependency relationships less explicit, and also makes them difficult to provide type inference for.

# Alternatives

There is currently a stage 1 proposal for adding the [Pipeline Operator](https://github.com/tc39/proposal-pipeline-operator) to JavaScript, which provides largely similar syntactical convenience:

``` js
let transformedMsg = msg |> uppsercase |> reverse |> pluralize
```

Considering there is a possibility of this proposal eventually landing, it is best for a framework like Vue to not provide a similar alternative (especially with syntax that conflicts with existing JavaScript).

That said, the proposal is still at stage 1 and hasn't received updates for quite a while, so it is not entirely clear whether it will end up landing, or whether it will land as it is designed right now. It is risky for Vue to adopt it as part of the official API, since we would be forced to introduce breaking changes if the spec ends up changing.

On the technical side, if we want to enable pipeline operators in Vue templates:

- Since template expressions are parsed using [Acorn](https://github.com/acornjs/acorn), we will need Acorn to support parsing pipeline operators. This should be possible via [Acorn plugins](https://github.com/acornjs/acorn#plugin-developments).

- Generated render function can be piped through Babel (with [appropriate plugins](https://babeljs.io/docs/en/babel-plugin-proposal-pipeline-operator)) for actual transpilation.

# Adoption strategy

Filters can be supported in a 2.x compatibility build, with warnings to guide progressive migration.
