- Start Date: 2020-01-21
- Target Major Version: 2.x, 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Provide more consistent custom CSS extensions in Single File Component scoped styles.

# Basic example

``` html
<style scoped>
/* deep selectors */
::v-deep(.foo) {}

/* targeting slot content */
::v-slotted(.foo) {}

/* one-off global rule */
::v-global(.foo) {}
</style>
```

# Motivation

Vue SFC scoped styles makes the CSS apply to the current component only. There are a number of cases that users commonly run into that can be improved.

## Deep Selectors

Sometimes we may want to explicitly make a rule target child components.

Originally we supported the `>>>` combinator to make the selector "deep". However, some CSS pre-processors such as SASS has issues parsing it since this is not an official CSS combinator.

We later switched to `/deep/`, which was once an actual proposed addition to CSS (and even natively shipped in Chrome), but later dropped. This caused confusion for some users, since they worry that using `/deep/` in Vue SFCs would make their code not supported in browsers that have dropped the feature. However, just like `>>>`, `/deep/` is only used as a compile-time hint by Vue's SFC compiler to rewrite the selector, and is removed in the final CSS.

To avoid the confusion of the dropped `/deep/` combinator, we introduced yet another custom combinator, `::v-deep`, this time being more explicit that this is a Vue-specific extension, and using the pseudo-class syntax so that any pre-processor should be able to parse it.

The previous versions of the deep combinator are still supported for compatibility reasons in the current [Vue 2 SFC compiler](https://github.com/vuejs/component-compiler-utils), which again, can be confusing to users. In v3, we are deprecating the support for `>>>` and `/deep/`.

As we were working on the new SFC compiler for v3, we noticed that CSS pseudo classes are in fact semantically NOT [combinators](https://developer.mozilla.org/en-US/docs/Learn/CSS/Building_blocks/Selectors/Combinators). It is more consistent with idiomatic CSS for pseudo classes to accept arguments instead, so we are also making `::v-deep()` work that way. The current usage of `::v-deep` as a combinator is still supported, however it is considered deprecated and will raise a warning.

## Targeting / Avoiding Slot Content

Currently, slot content passed in from the parent are affected by both the parent's scoped styles AND the child's scoped styles. There is no way to author rules that explicitly target slot content only, or ones that do not affect slot content.

In v3, we intend to make child scoped styles NOT affecting slot content by default. To explicitly target slot content, the `::v-slotted()` pseudo class can be used.

## One-Off Global Rules

Currently to add a global CSS rule we need to use a separate unscoped `<style>` block. We are introducing a new `::v-global()` pseudo class for one-off global rules.

---

> We are also aware that many users want to be able to use component props or state in the CSS of single-file components. We do have plans to support that but it will be in a separate RFC.

# Detailed design

- `>>>` and `/deep/` support are deprecated.

- `::v-deep` usage as a combinator is deprecated:

  ``` css
  /* DEPRECATED */
  ::v-deep .bar {}
  ```

  Instead, use it as a pseudo class that accepts another selector as argument:

  ``` css
  ::v-deep(.bar) {}
  ```

  The above compiles to:

  ``` css
  [v-data-xxxxxxx] .bar {}
  ```

- Slot content passed in from the parent no longer gets affected by child scoped styles by default. Instead, the child now needs to use the new `::v-slotted()` pseudo class to specifically target slot content:

  ``` css
  ::v-slotted(.foo) {}
  ```

  Compiles to:

  ``` css
  .foo[v-data-xxxxxxx-s] {}
  ```

  Notice the `-s` postfix which makes it target slot content only.

- New pseudo class `::v-global()` can be used to apply global rules inside a `<style scoped>` block:

  ``` css
  ::v-global(.foo) {}
  ```

  Compiles to:

  ``` css
  .foo {}
  ```

- The [test cases of `@vue/compiler-sfc`](https://github.com/vuejs/vue-next/blob/master/packages/compiler-sfc/__tests__/compileStyle.spec.ts) can be used as a reference for the compilation transform details.

# Adoption strategy

All previous usage of deep selectors can be supported with deprecation warnings. We will remove the support in a future release when most users have completed the migration.

The slot content behavior change should technically lead to more decoupled styling between parent and child components, however the behavior can indeed break existing styles that rely on child styles applied to slot content. We may need to provide an option to preserve the old behavior.

# Unresolved questions

We are not sure about how much the slot styling behavior change would affect existing components - any feedback would be appreciated.
