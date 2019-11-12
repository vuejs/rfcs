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

# Adoption strategy

Filters can be supported in a 2.x compatibility build, with warnings to guide progressive migration.
