- Start Date: 2019-03-28
- Target Major Version: 3 (2 is possible)
- Reference Issues: TBD
- Implementation PR:

# Summary

Controlling framing markup around (function-based) slots is hard or impossible
to manage in template based rendering. This RFC proposes new (compiler-level)
components to allow clear and concise handling of advanced slot usage scenarios.

# Basic example

```html
<template>
  <div class="wrapper">
    <slot>
      <div class="frame">
        <slot-content>
          Fallback content
        </slot-content>
      </div>
    </slot>
  </div>
</template>
```

```html
<template>
  <my-component>
    Custom content
    <slot-fallback />
  </my-component>
<template>
```

# Motivation

With the new slot syntax and behavior introduced in Vue 2.6 and the planned
removal of legacy slot behavior in Vue 3, in template based components it
becomes much harder to control optional slot rendering, slot fallback content
and framing markup around slots, that should not be rendered when slot content
is empty.

The following use cases are of interest:

- Provide framing markup in the slot defining component around non-empty slot
  content.
- Control the visibility of fallback slot content from within the slot
  content providing component.
- Add content to a slot while also displaying the fallback content.

When using custom render functions these use cases can be solved.

# Detailed design

## Slot frame

### Slot definition
```html
<template>
  <div class="wrapper">
    <slot>
      <div class="frame">
        <slot-content />
      </div>
    </slot>
  </div>
</template>
```
```js
function render(props, slots) {
  function defaultSlotFrame(content) {
    return h('div', { class: 'frame' }, content);
  }

  const defaultSlotContent = slots.default ? slots.default() : undefined;

  const defaultSlot = IS_EMPTY(defaultSlotContent)
    ? undefined
    : defaultSlotFrame(defaultSlotContent);

  return h('div', { class: 'wrapper' }, defaultSlot);
}
```

### Slot content provider

*No Changes*

## Slot fallback

### Slot definition
```html
<template>
  <div class="wrapper">
    <slot>
      Fallback content
    </slot>
  </div>
</template>
```
```js
function render(props, slots) {
  function defaultSlotFallbackContent() {
    return 'Fallback content';
  }

  const defaultSlot = slots.default
    ? slots.default({}, defaultSlotFallbackContent)
    : defaultSlotFallbackContent();

  return h('div', { class: 'wrapper' } }, defaultSlot);
}
```

### Slot content provider

```html
<template>
  <my-component>
    My Content
    <slot-fallback />
  </my-component>
</template>
```
```js
function render() {
  function defaultSlot(slotProps, slotFallback) {
    return ['My Content', slotFallback()];
  }

  return h('my-component', { slots: { default: defaultSlot } });
}
```

## Combined

### Slot definition
```html
<template>
  <div class="wrapper">
    <slot>
      <div class="frame">
        <slot-content>
          Fallback content
        </slot-content>
      </div>
    </slot>
  </div>
</template>
```
```js
function render(props, slots) {
  function defaultSlotFallbackContent() {
    return 'Fallback content';
  }

  function defaultSlotFrame(content) {
    return h('div', { class: 'frame' }, content);
  }

  const defaultSlotContent = slots.default
    ? slots.default({}, defaultSlotFallbackContent)
    : defaultSlotFallbackContent();

  const defaultSlot = IS_EMPTY(defaultSlotContent) ? undefined : defaultSlotFrame(defaultSlotContent);

  return h('div', { class: 'wrapper' }, defaultSlot);
}
```

### Slot content provider

*Same as in the "Slot fallback" section*

# Drawbacks

## Slot frames

None.

## Slot fallback reference

Possible BC breaks. See "Unresolved Questions" section.

# Alternatives

Use custom render functions for higher control over slot rendering.

# Adoption strategy

This RFC just introduces new components or framework elements that can be
incrementally adopted in existing code bases as well as by developers already
familar with the usage of slots in Vue.

The BC break regarding fallback content (see "Unresolved Questions" section)
could be controlled by a global flag or maybe a per usage modifier
(`v-slot.fallback-if-empty`) or both, to allow a fluid migration. Alternatively
the current behavior can be preserved and the current workaround to avoid
fallback content can be used:
```html
<template>
  <my-component>
    <span v-if="false">My content</span>
    <!-- Never render fallback content -->
    <span v-else></span>
  </my-component>
</template>
```

# Unresolved questions

- How to deal with nested slot fallback reference?
  ```html
  <template>
    <my-component>
      <my-other-component>
        My Content
        <!-- Which components slot fallback content is referenced? -->
        <slot-fallback />
      </my-other-component>
    </my-component>
  </template>
  ```

- The equivalent render functions provided in the "Detailed design" section
  introduce a BC break regarding conditional slot content:

  When slot content is provided, that means if a slot function is generated, the
  fallback content is never rendered implicitly, even if the return value of the
  generated slot function is empty.

  Example:
  ```html
  <template>
    <my-component>
      <span v-if="false">My Content</span>
    </my-component>
  </template>
  ```
  This would never display the slot fallback content. To get the current
  behavior:
  ```html
  <template>
    <my-component>
      <span v-if="false">My Content</span>
      <slot-fallback v-else />
    </my-component>
  </template>
  ```
