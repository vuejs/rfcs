- Start Date: 2019-03-12
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Unify the concepts of normal vs. scoped slots. They are all just slots in v3.

# Motivation

- The separation of normal vs. scoped slots is a result of scoped slots being added on later as a new concept, and they have different internal implementations in 2.x. However, this separation is technically unnecessary. Unifying the two can simplify the overall concept of slots.

- Component authors using render functions no longer need to worry about handling both  `$slots` and `$scopedSlots`.

- Compiling all slots as functions leads to better performance in large component trees.

  Quote from 2.6 release announcement:

  > Normal slots are rendered during the parent’s render cycle. When a dependency of a slot changes, it causes both the parent and child components to re-render. Scoped slots, on the other hand, are compiled into inline functions and called during the child component’s render cycle. This means any data dependencies relied on by a scoped slot are collected by the child component, resulting in more precise updates. In 2.6, we have introduced an optimization that further ensures parent scope dependency mutations only affect the parent and would no longer force the child component to update if it uses only scoped slots.

# Detailed design

Slots unification involves two parts:

- Syntax unification (shipped in 2.6 as `v-slot`)

- Implementation unification: compile all slots as functions.

  - `this.$slots` now exposes slots as functions;

  - `this.$scopedSlots` removed.

  - In 2.x, all slots using `v-slot` syntax are already compiled as functions internally. `this.$scopedSlots` also proxies to normal slots and expose them as functions.

## Usage in Render Functions

Existing render function usage will remain supported. When passing children to a component, both VNodes and functions are supported:

``` js
h(Comp, [
  h('div', this.msg)
])

// equivalent:
h(Comp, () => [
  h('div', this.msg)
])
```

Inside `Comp`, `this.$slots.default` will be a function in both cases and returns the same VNodes. However, the 2nd case will be more performant as `this.msg` will be registered as a dependency of the child component only.

Named slots usage has changed - instead of a special `slot` data property on the content nodes, pass them as children via the 3rd argument:

``` js
// 2.x
h(Comp, [
  h('div', { slot: 'foo' }, this.foo),
  h('div', { slot: 'bar' }, this.bar)
])

// 3.0
// Note the `null` is required to avoid the slots object being mistaken
// for props.
h(Comp, null, {
  foo: () => h('div', this.foo),
  bar: () => h('div', this.bar)
})
```

### Further Manual Optimization

Note that when the parent component updates, `Comp` is always forced to also update because without the compile step Vue doesn't have enough information to tell whether `slots` may have changed.

The compiler can detect `v-slot` and compile content as functions, but in render functions this won't happen automatically. We may also be able to perform similar optimizations in our JSX babel plugin. But for users writing direct render functions, they need to do it manually in performance sensitive use cases.

Slots can be manually annotated so that Vue won't force the child to update when the parent updates:

``` js
h(Comp, null, {
  $stable: true,
  foo: () => h('div', this.foo),
  bar: () => h('div', this.bar)
})
```

# Adoption strategy

Majority of the change has already been shipped in 2.6. The only thing left is the removal of `this.$scopedSlots` from the API. In practice, the current `this.$scopedSlots` in 2.6 works exactly like `this.$slots` in 3.0, so the migration can happen in 2 steps:

1. Use `this.$scopedSlots` everywhere in the 2.x codebase;
2. Replace all `this.$scopedSlots` occurrences with `this.$slots` in 3.0.
