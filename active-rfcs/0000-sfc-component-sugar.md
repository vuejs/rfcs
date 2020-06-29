- Start Date: 2020-06-29
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Syntax sugar for reducing component import and registration boilerplate in Single File Components.

# Basic example

```html
<component src="./Foo.vue"/>
<component async src="./Bar.vue"/>
<component src="./Baz.vue" as="Qux" />

<template>
  <Foo/>
  <Bar/>
  <Qux/>
</template>
```

# Motivation

Currently in SFCs you have to import a component and then pass it to the `export default { components: { ... } }` hash, which leads to a lot of redundancy: for a single component we are repeating its name 3 times: in the imported binding, the file name, and the components option.

The `<component/>` sugar requires the name of each component to be specified only once.

# Detailed design

## Normal Components

**Before**

```html
<template>
  <Foo/>
</template>

<script>
import Foo from './Foo.vue'

export default {
  components: {
    Foo
  }
}
</script>
```

**After**

```html
<component src="./Foo.vue"/>

<template>
  <Foo/>
</template>
```

## Async Components

**Before**

```html
<template>
  <Foo/>
</template>

<script>
import { defineAsyncComponent } from 'vue'

export default {
  components: {
    Foo: defineAsyncComponent(() => import('./Foo.vue'))
  }
}
</script>
```

**After**

```html
<component async src="./Foo.vue" />

<template>
  <Foo/>
</template>
```

## Component Renaming

By default, the component's locally registered name is inferred from its filename. But they can be renamed locally:

```html
<component src="./Foo.vue" as="Bar" />

<template>
  <Bar/>
</template>
```

# Drawbacks

This would require updates in tools that parse SFC content for template analysis - e.g. Vetur & `@vuedx`.

However, since this information is going to be provided directly by `@vue/compiler-sfc` in the parsed SFC descriptor, it should remove some extra complexity from these tools as well.

# Alternatives

We considered implicitly registering imported components:

```html
<template>
  <Foo/>
</template>

<script>
import Foo from './Foo.vue'
</script>
```

However, this approach has a few issues:

- `Foo` is unused in the script scope, making it annoying when using linter rules that check for unused variables.

- The only safe assumption about an import being a Vue component is the `.vue` extension, which makes it unable to support components authored in non-SFC formats (e.g. `import Foo from './Foo/ts'` can be a component but we can't really be sure).

# Adoption strategy

This is a fully backwards compatible new feature. However, we probably want to warn users against mixing the manual imports and `<component>` tags so that tools that rely on the extracted information can make safer assumptions.
