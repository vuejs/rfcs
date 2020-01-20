- Start Date: 2020-01-20
- Target Major Version: (3.x)
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR:

# Summary

- Adds a `<portal>` component to Vue core
- the component accepts a DOM selector via a prop
- the component moves its children to the element identified by the DOM selector
- At the virtual DOM level, the children stay descendants of the `<portal>` though, so they i.e. have access to injections from its ancestors

# Basic example

```html
<body>
  <div id="app">
    <h1>Move the #content with the portal component</h1>
    <portal target="#endofbody">
      <div id="content">
        <p>
          this will be moved to #endofbody.<br />
          Pretend that it's a modal
        </p>
        <Child />
      </div>
    </portal>
  </div>
  <div id="endofbody"></div>
  <script>
    const { Portal } = window.Vue;
    new Vue({
      el: "#app",
      components: {
        Portal,
        Child: { template: "<div>Placeholder</div>" }
      }
    });
  </script>
  <body></body>
</body>
```

This will result in the following behaviour:

1. All of the children of `<portal>` - in this example: `<div id="content">` - will be appended to `<div id="endofbody">`
2. the `<Child>` component as one of these children will remain a child component of the `<Portal>`'s parent (the Portal is transparent).

```html
<div id="app">
  <!-- -->
</div>
<div id="endofbody">
  <div id="content">
    <p>
      this will be moved to #endofbody.<br />
      Pretend that it's a modal
    </p>
    <div>Placeholder</div>
  </div>
</div>
```

# Motivation

Vue encourages us to build our UIs by encapsulating UI and related behaviour into components, which we can nest inside one another to build a tree of components that make up your application UI. That model has proven itself in Vue and other frameworks in many ways, but there's one weakness this RFC seeks to address:

Sometimes, a part of a component's template belongs into this component _logically_, while from a technical point of view (i.e.: styling requirements), it would be preferable to move this part of the template somewhere else in the DOM, breaking it out of it's deeply nested position without our DOM tree.

## Use cases

### z-Index

The main use cases for such a behaviour are usually styling-related. Various common UI patterns such as modals, dialogs, dropdown menus, notifications etc. require fixed or absolute positioning and management of their z-index.

In order to work around issues with [z-index Stacking Context](https://philipwalton.com/articles/what-no-one-told-you-about-z-index/) behaviour, it's a common pattern to put the DOM elements of those components right before the `</body>` tag in order to move them out of any parent element's z-index stacking context.

### Widgets

Many apps have the concept of widgets, where their UI has an outlet (i.e. in a sidebar or dashboard) where other parts of the application, i.e. plugins, can inject small pieces of UI.

In Single Page Applications, where our Javascript controls essentially he whole page, this is generally not a challenge. But in situations where our Vue app only controls a part of the page, it currently proves to be challenging (but impossible) to mount individual elements and components in other parts of the page.

With Portals, we have a straightforward way to mount child components to other locations in the DOM declaratively.

# Detailed design

## Globally imported `Portal` component

The `'vue'` package has a named export for a `<Portal>` "component".

Since this component doesn't have any component logic of its own (Portal functionality would be implemented at the virtual DOM level), so this could be a `Symbol` instead of a full component.

```js
import { Portal } from "vue";
export default {
  template: `<div>
    <portal target="#endofbody">
      Some content.
    </portal>
  <div>`,
  components: {
    Portal
  }
};
```

When using a render function, the component can be used directly without first registering it, like any other component:

```js
import { Portal, h } from "vue";
export default {
  render() {
    return h("div", [h(Portal, { target: "#endofbody" }, ["Some content"])]);
  },
  // or with JSX:
  render() {
    <div>
      <Portal target="#endofbody">Some content</portal>
    </div>;
  }
};
```

If used directly in the browser from a CDN, we can access the Portal as a property on the Vue constructor:

```js
const App = {
  template: `<div>
    <portal target="#endofbody">
      Some content.
    </portal>
  <div>`,
  components: {
    Portal: Vue.Portal
  }
};
```

## The `target` prop

The component has only one _required_ prop, named `target`. It accepts a string wich has to be a valid query selector.

```html
<!-- ok -->
<Portal target="#some-id">
  <Portal target=".some-class">
    <Portal target="[data-portal]">
      <!-- 
  probably too unspecific, but technically valid 
  should we allow this or block it?
-->
      <Portal target="h1">
        <!-- Wrong -->
        <Portal target="some-string"></Portal></Portal></Portal></Portal
></Portal>
```

## Lifecycle

### Mounting

When the Portal component is mounted by its parent, it will use the `target` prop's value as a selector.

- If the query returns an element, the slot children of the `Portal` will be mounted as child nodes of that element in the DOM
- If this element doesn't exist in the DOM at the moment that this `<Portal>` is mounted, a warning will be logged during development (nothing would happen in production):

```js
`Portal could not be mounted to element with selector '${props.target}': element not found in DOM.

// following would be a display where in the component tree this happened etc.
```

#### \$parent

If the children of `Portal` contain any components, their `this.$parent` property should reference the `Portal`'s parent component. In other words, these components stay in their original spot in the _component tree_, even though they ended up mounted somewhere else in the _DOM tree_.

`Portal`, not being a real component at all, is transparent and will not appear as an ancestor in the `$parent`chain.

```html
<template>
  <Portal v-bind:target="targetName">
    <Child />
  </Portal>
  <template>
    <script>
      import { Portal } from 'vue'
      export default {
        name: 'Parent'
        components: {
          Portal,
          Child: {
            template: '<div/>',
            mounted() {
              console.log(this.$parent.$options.name )
              // => 'Parent'
            }
          }
        },
      }
    </script></template
  ></template
>
```

Similarly, using `inject` in `Child` should be able to inject any provided content from `Paren` or one of its ancestors.

### Updating

The `target` prop can be changed dynamically with `v-bind`. When the value changes, `Portal` will remove the children from the previous target and move them to the new one.

If the children contain any component instances, these will not be influenced by this. The instances will be kept alive, keep their state etc.

```html
<template>
  <Portal v-bind:target="targetName">
    <p>This can be moved around with the button below</p>
  </Portal>
  <button v-on:click="toggleTarget">Toggle</button>
  <hr />
  <div id="A"></div>
  <div id="B"></div>
  <template>
    <script>
      import { Portal } from "vue";
      export default {
        components: { Portal },
        data: () => ({
          targetName: "A"
        }),
        methods: {
          toggleTarget() {
            this.targetName = this.targetName == "A" ? "B" : "A";
          }
        }
      };
    </script></template
  ></template
>
```

If the new target selector doesn't match any elements, a warning should be logged during development.

> **Question:** Should the old content still be removed from the old target in that situation or should everything stay the way it is?

> **Question** What should happen when the parent component re-renders? should the query Selector be run again? Seems like it's unnecessary in most situations, but if we don't, then having a selector that doesn't match anything initially will result in a stale component even if that element pops up later, wouldn't it?

### Destruction

When a `Portal` is being destroyed (e.g. because its parent component is being destroyed or because of a `v-if`), its children are removed from the DOM and any component instances destroyed just like they were still children iof the parent.

### dev-tools

The Portal should not appear in the chain of parent components (`this.$parent`), but it should be identifiable within the virtual DOM so that Vue's dee-tools can show them in their visualisation of the component tree.

### Using a Portal on an element within a Vue app

Technically, this proposal allows to select _any_ element in the DOM , including elements that are rendered by our Vue app in some other part of the component tree.

But that puts the portal'd slot content under the control of that other component's lifecycle, which means the content can possibly be removed from the DOM if that component gets destroyed.

Any component that came through a `Portal` would effectively have its DOM removed by still be in the original virtual DOM tree, which would lead to patch errors when these components tried to update.

# Drawbacks

The only notable drawback that we see is the additional code required to implement this. But judging from experiments in the prototype, that code will be very light, as it's just a slightly different way to mount elements at the virtualDOM level.

As it's an additive feature and the functionality is pretty straightforward (one prop defining as target selector), this should also not add much complexity to Vue in terms of documentation or teaching.

When considering how popular current userland solutions are even with their caveats and limitations, the cost/benefit ratio seems clear.

# Alternatives

No other designs were considered so far.

## What happens if we don't do this

There are currently several userland implementations of this feature available, which usually suffer some caveats and drawbacks that stem of the fact that portals are not supported at the virtual DOM level in Vue 2.

People could continue to use these with their existing limitations and drawbacks.

# Adoption strategy

Portals is a new feature and as such purely additive in nature.

Since the component is not globally registered (unlike e.g. `<transition>` was in Vue 2.0), there are also not risks if naming conflicts for applications that already use the name `<portal>` in some other context:

```html
<template>
  <portal> <!-- custom component --></portal>
  <VuePortal><!-- portal from this RFC --></VuePortal>
  <template>
    <script>
      import { Portal } from "vue";
      export default {
        components: {
          VuePortal: Portal
        }
      };
    </script></template
  ></template
>
```

As such, this feature does not have any impact on the migration of Vue 2.0 applications to Vue 3.0.

Users new to Vue 3.0 or Vue in general will be able to learn about this feature from the docs in the usual way and gradually introduce it into their projects where it makes sense.

## Existing 3rd party solutions

As mentioned, several 3rd party plugins/libs implement similar functionality right now.

Some of them may become irrelevant though this RFC, while others, offering functionality that exceeds what this proposal describes, would could adapt their implementation to make use of this proposal's "native" `<portal>` component internally.

If RFC vuejs/vue-next#28 (Render function change) is adopted, these libraries will have to be reworked either way, at which point they can adopt this new feature.

# Unresolved questions

### Mounting to elements controlled by Vue

When portal-ing content to a target element that is controlled by Vue, the portal's content might be removed from the DOM by the component that controls that target element.

- Can we handle this gracefully?
- If not: Are userland solutions on top of this basic implementations able to work around this (in my opinion: yes)
- Or is this feature strictly limited to mounting to elements _outside_ of the part of the DOM controlled by Vue?

### Missing Targets

- What do we do if the new target doesn't exist? unmount the old one anyway?
- Should target selectors re-run on every re-render of the `Portal`'s parent so that initially missing targets can be mounted to once they exist?

### using multiple portals on the same target

portal-vue supports sending content from multiple `<portals>` to the same target. it does so by using a `<portal-target>` component that manages this.

The portal functionality proposed in this RFC requires that each portal has its own target element to mount to.

Should we and if so how can we make n:1 work?

### Naming conflict with native portals

There's proposal for native portals:

- Spec: https://wicg.github.io/portals/
- Introduction: https://web.dev/hands-on-portals/

We probably don't want to have a naming conflict with a future HTML element that may be called `<portal>`, especially since it's functionality is about something completely different form what portals in libs like Vue or React mean right now.

So should we give the concept of this RFC (and by extension, the component it introduces) another name to prepare for native `<portal>` elements becoming a standard? If so, what would we call it instead?

- `<Teleport>`
- `<Wormhole>`
- `<...?>`

Or should we keep it as the concept of what a portal is in Vue, React e.t al. is already "common knowledge" and a new term might confuse people more than it would help?
