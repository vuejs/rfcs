- Start Date: 2026-07-09
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR:

# Summary

Introduce a compiler-reserved `<Self>` template tag that renders the current component. This allows recursive components to refer to themselves without depending on the component's `name` option or an SFC filename-derived inferred name.

# Basic example

```vue
<!-- TreeNode.vue -->
<script setup lang="ts">
interface TreeNode {
  id: string
  label: string
  children?: TreeNode[]
}

defineProps<{
  node: TreeNode
}>()
</script>

<template>
  <li>
    <span>{{ node.label }}</span>

    <ul v-if="node.children?.length">
      <Self
        v-for="child in node.children"
        :key="child.id"
        :node="child"
      />
    </ul>
  </li>
</template>
```

If `TreeNode.vue` is later renamed to `NestedListItem.vue`, the recursive reference does not need to be renamed because it is no longer expressed as `<TreeNode>`.

# Motivation

Today, a recursive component must refer to itself by name. With the Options API, this usually means declaring a `name` option and using that name in the template:

```vue
<script>
export default {
  name: 'TreeNode',
  props: {
    node: Object
  }
}
</script>

<template>
  <TreeNode :node="child" />
</template>
```

With SFCs, Vue can infer a component's name from the filename, so `TreeNode.vue` can recursively reference itself as `<TreeNode />`. This avoids an explicit `name` option, but it still couples the recursive reference to a name derived from the file path.

That coupling creates a maintenance hazard:

- Renaming a file requires updating every recursive self-reference in the component template.
- It is easy to forget that the filename is also part of the component's recursive API.
- The recursive reference can conflict with an imported or locally registered component of the same name.
- `name` is also used for DevTools, warning traces, and `<KeepAlive>` include / exclude matching, but those display and matching concerns are separate from the semantic intent to render "this component again."

Recursive components are common for trees, nested menus, threaded comments, document outlines, AST viewers, and schema-driven renderers. In these cases, the intent is local and structural: render another instance of the same component. The template should be able to express that intent directly.

# Detailed design

## `<Self>` in templates

`<Self>` is a special component tag recognized by Vue's template compiler. It always resolves to the component currently being rendered by the template in which the tag appears.

The reserved spelling is exactly `<Self>` in case-preserving templates such as SFC templates and string templates. The lowercase `<self>` spelling is not special in those templates.

```vue
<template>
  <Self v-if="hasMore" :node="nextNode" @select="onSelect" />
</template>
```

`<Self>` behaves like a normal component invocation:

- props, attrs, events, directives, `v-if`, `v-for`, `v-show`, and slots work the same way as they do on any component tag.
- a `ref` placed on `<Self>` refers to the child instance created by that tag, not the parent instance.
- it creates a new child component instance. It is not an inline expansion of the current template.
- recursion still requires the user to provide a terminating condition, exactly like name-based recursive components do today.

For example:

```vue
<template>
  <Self v-if="node.parent" :node="node.parent">
    <template #default="{ depth }">
      {{ depth }}
    </template>
  </Self>
</template>
```

## Resolution semantics

`<Self>` is reserved by the compiler and has higher priority than local component bindings, explicit component registrations, and global component registrations.

This means the following template renders the current component, not the imported `Self` binding:

```vue
<script setup>
import Self from './OtherComponent.vue'
</script>

<template>
  <Self />
</template>
```

If an imported or locally registered component named `Self` is needed, it should be renamed and used through that other name:

```vue
<script setup>
import OtherSelf from './OtherComponent.vue'
</script>

<template>
  <OtherSelf />
</template>
```

The compiler should emit a development warning when a component binding named `Self` is present in the same template scope, because the binding will be shadowed by the reserved tag.

## Compiler output

The compiler can lower `<Self>` to an internal helper that resolves the current component type from the current rendering instance.

Conceptually:

```js
import {
  createVNode as _createVNode,
  resolveSelfComponent as _resolveSelfComponent
} from 'vue'

export function render(_ctx, _cache) {
  const _Self = _resolveSelfComponent()
  return _createVNode(_Self, { node: _ctx.child })
}
```

The exact helper name is not part of the public API. The important behavior is that the rendered vnode uses the same component definition as the component whose template contains `<Self>`.

The same lowering should be supported by both the DOM compiler and the SSR compiler so that client-side rendering and server-side rendering produce equivalent component trees.

## Relationship with component `name`

This proposal does not remove or change component name inference.

The `name` option and SFC filename-based inferred names remain useful for:

- DevTools inspection.
- warning component traces.
- `<KeepAlive>` include / exclude matching.
- existing recursive component references.

`<Self>` only provides a name-independent way to express recursive self-reference in templates.

## Supported authoring forms

`<Self>` is intended to work in normal SFC templates, including SFCs using `<script setup>`:

```vue
<script setup>
defineProps({
  item: Object
})
</script>

<template>
  <Self v-for="child in item.children" :key="child.id" :item="child" />
</template>
```

It should also work in templates authored with the Options API:

```vue
<script>
export default {
  props: {
    item: Object
  }
}
</script>

<template>
  <Self v-for="child in item.children" :key="child.id" :item="child" />
</template>
```

Render functions and JSX are out of scope for this proposal because they can already close over the current component definition explicitly when needed.

## Tooling

Vue-aware tooling should treat `<Self>` as a built-in special component:

- language tools should not report it as an unresolved component.
- auto-import tooling should not generate an import for it.
- `eslint-plugin-vue` should be able to recognize it as a valid component tag.
- formatters should preserve it like any other PascalCase component tag.

# Drawbacks

`Self` becomes a reserved component tag. Existing applications that intentionally use a local or global component named `Self` would need to rename that component or render it dynamically.

The feature adds compiler and tooling complexity for a relatively small ergonomic improvement. The implementation is likely modest, but support needs to be consistent across the SFC compiler, runtime compiler, SSR compiler, language tools, and lint rules.

The feature introduces another special template tag. Users need to learn that `<Self>` is not imported, registered, or derived from the filename.

It does not eliminate all uses of component names. Components may still need names for DevTools, warning traces, and `<KeepAlive>` matching.

# Alternatives

## Keep relying on `name` or filename inference

This is the current behavior. It avoids adding a new special tag, but keeps recursive self-reference coupled to component names.

## Use `defineOptions({ name })`

`defineOptions()` can make the component name explicit in `<script setup>`, but the template still needs to repeat the same name:

```vue
<script setup>
defineOptions({
  name: 'TreeNode'
})
</script>

<template>
  <TreeNode />
</template>
```

This is clear, but it keeps two declarations in sync.

## Use a compiler macro

A macro such as `defineSelf()` could expose the current component as a binding:

```vue
<script setup>
const Self = defineSelf()
</script>

<template>
  <component :is="Self" />
</template>
```

This avoids reserving a tag, but it is more verbose and pushes a purely template-level concern into the script block.

## Rely on lint rules and rename tooling

Tooling could warn when a recursive tag no longer matches the component filename or explicit name. This would reduce mistakes, but it would not remove the underlying coupling.

# Adoption strategy

This is an additive feature. Existing recursive components continue to work unchanged.

Developers can adopt `<Self>` incrementally in recursive components:

```diff
- <TreeNode :node="child" />
+ <Self :node="child" />
```

A codemod could be provided for simple SFC cases where a component recursively references the inferred filename or explicit `name`, and where no local binding conflict exists. Because component resolution can involve local and global registrations, such a codemod should be conservative.

Documentation should update the recursive component sections to recommend `<Self>` when the intent is to render another instance of the current component, while documenting that `name` remains relevant for diagnostics and `<KeepAlive>`.

# Unresolved questions

Should `<Self>` be supported in in-DOM templates? Browser HTML parsing lowercases tag names before Vue sees them, so supporting in-DOM templates would likely require also reserving `<self>`, which has a higher compatibility risk.

Should Vue expose a public render-function helper for the same concept, or should this remain a template-only feature?

Should the compiler warn, error, or silently shadow when a local binding named `Self` exists?
