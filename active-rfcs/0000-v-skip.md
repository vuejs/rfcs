- Start Date: 2024-10-24
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

This RFC proposes the addition of a new directive, `v-skip`. The `v-skip` directive conditionally skips rendering the current element or component while preserving and promoting its child nodes to the parent level in the node hierarchy. This behavior differs from `v-if`, which entirely removes the node and its subtree when the condition is `false`.

# Basic example

```vue
<template>
  <Tooltip v-skip="!disabled" content="The button is disabled because ...">
    <button :disabled="disabled">Submit</button>
  </Tooltip>
</template>
```

# Motivation

There are scenarios where developers need to conditionally remove an element or component without discarding its children. Currently, achieving this requires workarounds such as using userland components or switching to render functions, which can be verbose and less intuitive.

Related:

- https://github.com/vuejs/vue/issues/12033

  This issue discussed the need of a conditional wrapper. The workaround (bare basic, not robust) provided in [one comments](https://github.com/vuejs/vue/issues/12033#issuecomment-1183981371) is:

  ```vue
  <template>
    <component :is="tag" v-if="wrap">
      <slot />
    </component>
    <slot v-else />
  </template>

  <script setup lang="ts">
  interface Props {
    tag: string;
    wrap: boolean;
  }

  defineProps<Props>();
  </script>
  ```

  There was also a [comment](https://github.com/vuejs/vue/issues/12033#issuecomment-1167564997) proposing a `v-wrap-if` directive:

  ```vue
  <div v-wrap-if="someCondition">
    <span>
      This is a span that is sometimes wrapped up in a div.
    </span>
  </div>
  ```

- https://github.com/vuejs/rfcs/pull/449

  This RFC propsed a built-in `fragment` component to make utilizing conditional wrappers easier.

  ```vue
  <template>
    <component :is="shouldWrap ? WrapperComponent : 'fragment'">
      <img src="cat.jpg" alt="Cat" />
    </component>
  </template>
  ```

- https://dev.to/alexander-nenashev/creating-a-conditional-wrap-component-in-vue-3-18ml

  This post series introduced a way to implement a conditional wrapper component which is quite cumbersome and relied on some knowledge of Vue's internals. The usage of the component is also not as concise as a directive.

The userland component is quite hard to be implemented correctly and when it comes to toggling a component and passing through its props, slots and event listeners, it becomes verbose to use as well. It becomes even harder if we want correct type support.

The `v-skip` directive aims to provide a more concise and declarative approach to tackle this problem. It eliminates the need for userland components, eases the pain of passing props
 or switching to render functions, and makes the code more readable and maintainable.

# Detailed design

The `v-skip` directive accepts an expression that evaluates to a boolean value.

## For elements

```vue
<div v-skip="condition">
  <p>Hello</p>
</div>
```

- If `condition` is truthy, the current node is skipped, and its children are rendered in its place (render as if it's a fragment node) and produces:

  ```vue
  <p>Hello</p>
  ``` 

- Otherwise, the node and its children are rendered normally and produces:

  ```html
  <div>
    <p>Hello</p>
  </div>
  ```

## For components

Let's say we have a `Tooltip` component which is used like this:

```vue
<Tooltip>
  <button :disabled="disabled">Submit</button>
  <template #content>
    <em>The button is disabled because ...</em>
  </template>
</Tooltip>
```

Which renders:

```html
<div>
  <button>Submit</button>
  <div class="tooltip">
    <em>The button is disabled because ...</em>
  </div>
</div>
```

We can conditionally skip the `Tooltip` component like this:

```vue
<Tooltip v-skip="!disabled">
  <button :disabled="disabled">Submit</button>
  <template #content>
    <em>The button is disabled because ...</em>
  </template>
</Tooltip>
```

- If `disabled` is falsy (`v-skip` evaluates to `true`), the `Tooltip` component is skipped, we directly render its `default` slot in its place and discard other slots, which produces:

```html
<button>Submit</button>
```

## Implementation details

- The children of the node with `v-skip` directive should be rerendered when the condition changes as the their parent may have provided contextual data (`provide`, etc.).
- `v-skip` cannot be used on `<template>` and `<slot>`.

# Drawbacks

- We may increase the Vue bundle size slightly due to the addition of the new directive.

# Alternatives

Alternatives in userland is already covered in the motivation section.

Regarding to the naming, we can also consider alternatives like:

- `v-skip-if`
- `v-wrap`
- `v-wrap-if`
- `v-unwrap`
- `v-unwrap-if`

# Adoption strategy

## Backward Compatibility

The addition of `v-skip` does not break existing code and is entirely opt-in.

## Documentation

Comprehensive documentation and examples will be provided to educate developers on proper usage.

## Tooling Support

We need built-in support from Vue Devtools and the official language support to recognize and correctly parse the `v-skip` directive. We also need to add linting rules to `eslint-plugin-vue` enforce proper usage of the directive.

# Unresolved questions

N/A
