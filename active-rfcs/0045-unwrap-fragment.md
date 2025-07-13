---
title: unwrapFragment to flatten nested Fragment nodes
start-date: 2025-07-01
type: Feature
status: In Progress
---

# Summary

Introduce an internal utility function `unwrapFragment` to flatten nested `Fragment` nodes inside slot VNode arrays. This helps simplify component logic and improve consistency when working with deeply nested slots.

# Motivation

In Vue 3, when a slot returns multiple root nodes, they are automatically wrapped in a `Fragment`. However, in real-world usage—especially in multi-level component compositions—slots passed from parent to child can accumulate multiple layers of `Fragment` wrappers.

This creates issues for component libraries or advanced slot processing scenarios where:

- Accurate length or type checking of slot children is needed
- Conditional logic based on vnode types fails (e.g. showing expand/collapse buttons)
- Tools like developer inspectors or snapshot serializers receive unnecessarily nested structures

## Real-world Example

In [Tencent's open source component library TDesign](https://github.com/Tencent/tdesign-vue-next), the `Alert` component determines whether to show a toggle button by counting the slot content. When a user wraps `Alert` in a custom component and forwards the slot, it ends up wrapped in a second-level `Fragment`, causing logic to fail and rendering to behave unexpectedly.

This problem has also been observed in internally-developed systems that rely on dynamic slot inspection or vnode traversal for layout logic.

# Detailed Design

```ts
// unwrapFragment.ts
import { Fragment, VNode } from '@vue/runtime-core'

export function unwrapFragment(vnodes: VNode[] | undefined): VNode[] {
  if (!vnodes) return []
  const result: VNode[] = []
  for (const vnode of vnodes) {
    if (vnode.type === Fragment) {
      result.push(...unwrapFragment(vnode.children as VNode[]))
    } else {
      result.push(vnode)
    }
  }
  return result
}
```

This utility:

- Accepts a VNode array
- Recursively flattens all `Fragment` nodes
- Returns the "real" vnode list used for actual render logic

This function is **internal-only** and meant to assist renderer and advanced component authors. It is not proposed as a public API.

# Alternatives Considered

- Manually flattening in userland: leads to duplicated logic and error-prone usage
- Using `.flatMap` or `.children`: does not handle nested Fragments recursively
- Opting out of Fragment wrapping: not currently possible and would break existing semantics

# Drawbacks

- Adds a small runtime utility to `@vue/runtime-core` (~0.1kb)
- Slight conceptual overhead (introducing another vnode helper)

# Adoption Strategy

- Used internally by component libraries
- Can be documented as part of advanced authoring guides
- May eventually be moved to `@vue/shared` if needed by other runtimes

# Unresolved Questions

- Should Vue provide a built-in way to unwrap slots by default?
- Should a `.raw()` helper be introduced on `slots.default`?

# Future Possibility

If this proves widely useful, Vue may consider exposing `unwrapFragment` as a public utility or embedding the behavior in slot normalization logic.
