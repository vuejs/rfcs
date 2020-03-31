- Start Date: 2020-01-20
- Target Major Version: (3.x)
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR:

# Summary

- Adds a `<teleport>` component to Vue core
- the component requires a target element, provided through a prop which expects an `HTMLElement` or a `querySelector` string.
- the component moves its children to the element identified by the DOM selector
- At the virtual DOM level, the children stay descendants of the `<teleport>` though, so they i.e. have access to injections from its ancestors

# Basic example

```html
<body>
  <div id="app">
    <h1>Move the #content with the portal component</h1>
    <teleport to="#endofbody">
      <div id="content">
        <p>
          this will be moved to #endofbody.<br />
          Pretend that it's a modal
        </p>
        <Child />
      </div>
    </teleport>
  </div>
  <div id="endofbody"></div>
  <script>
    new Vue({
      el: "#app",
      components: {
        Child: { template: "<div>Placeholder</div>" }
      }
    });
  </script>
</body>
```

This will result in the following behaviour:

1. All of the children of `<teleport>` - in this example: `<div id="content">` and `<Child />` - will be appended to `<div id="endofbody">`
2. the `<Child>` component as one of these children will remain a child component of the `<teleport>`'s parent (the `<teleport>` is transparent).

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

In Single Page Applications, where our Javascript controls essentially the whole page, this is generally not a challenge. But in situations where our Vue app only controls a part of the page, it currently proves to be challenging (but impossible) to mount individual elements and components in other parts of the page.

With `<teleport>`, we have a straightforward way to mount child components to other locations in the DOM declaratively.

# Detailed design

## Implementation as an internal component

The `<teleport>` "component" is an internal component like `<transition>` and `<keep-alive>`. It's tree-shakable, so if you don't use the feature, the component code will be dropped from the final bundle.

### Usage in templates

In templates, the compiler will add the import for the `<teleport>` component to the generated code, so it can be used just like this:

```js
export default {
  template: `
    <div>
      <teleport to="#endofbody">
        Some content.
      </teleport>
    <div>
  `
};
```

When using a render function or JSX, component has to imported first, like any other component:

```js
import { Teleport, h } from "vue";
export default {
  render() {
    return h("div", [h(Teleport, { target: "#endofbody" }, ["Some content"])]);
  },
  // or with JSX:
  render() {
    <div>
      <Telport target="#endofbody">Some content</Teleport>
    </div>;
  }
};
```

## using multiple portals on the same target

A common use case scenario would be a reusable `<Modal>` component of which there might be multiple instances active at the same time. For this kind of scenario, multiple `<teleport>` components can mount their content to the same target element. The order will be a simple `append` - later mounts will be located after earlier ones within the target element.

```html
<teleport to="#modals">
  <div>A</div>
</teleport>
<teleport to="#modals">
  <div>B</div>
</teleport>

<!-- result-->
<div id="modals">
  <div>A</div>
  <div>B</div>
</div>
```

In the discussions for this RFC so far, more complex behaviour (optional prepending, defining the order ...) was discussed, but concerns about complexity and foreseeable issues with SSR and hydration lead us to limit this behaviour to a simple `append`.

## Props

### `to`

The component has only one _required_ prop, named `to`. It accepts a string wich has to be a valid query selector, or an HTMLElement (if used in a browser environment).

```html
<!-- ok -->
<teleport to="#some-id" />
<teleport to=".some-class" />
<teleport to="[data-portal]" />
<!--
  probably too unspecific, but technically valid
  should we allow this or block it?
-->
<teleport to="h1" />
<!-- Wrong -->
<teleport to="some-string" />
></teleport>
```

### `disabled`

This optional prop can be used to disable the portal's functionality, which means that its slot content will not be moved anywhere and instead be rendered where you specified the `<teleport>` in the surrounding parent component.

```html
<teleport to="#popup" :disabled="displayVideoInline">
  <video src="./my-movie.mp4">
</teleport>
```

Changing its value dynamically allows to move the same DOM elements between the target specified by the `to` prop, and the actual location in the surrounding parent component. This means that any components inside of the `<teleport>` will be kept alive and keep their internal state. Likewise, a `<video>` element will keep its playback state while being moved betweeen these locations.

## Lifecycle

### Mounting

When the `<teleport>` component is mounted by its parent, it will use the `to` prop's value as a selector.

- If the query returns an element, the slot children of the `<teleport>` will be mounted as child nodes of that element in the DOM
- If this element doesn't exist in the DOM at the moment that this `<teleport>` is mounted, a warning like the following would be logged during development (nothing would happen in production):

```js
`Teleport content could not be mounted to element with selector '${props.to}': element not found.

// following would be a display where in the component tree this happened etc.
```

#### \$parent

If the children of `<teleport>` contain any components, their `this.$parent` property should reference the `<teleport>`'s parent component. In other words, these components stay in their original spot in the _component tree_, even though they ended up mounted somewhere else in the _DOM tree_.

`<teleport>`, not being a real component at all, is transparent and will not appear as an ancestor in the `$parent`chain.

```html
<template>
  <teleport v-bind:to="targetName">
    <Child />
  </teleport>
</template>
<script>
  export default {
    name: 'Parent'
    components: {
      Child: {
        template: '<div/>',
        mounted() {
          console.log(this.$parent.$options.name )
          // => 'Parent'
        }
      }
    },
  }
</script>
```

Similarly, using `inject` in `Child` should be able to inject any provided content from `Paren` or one of its ancestors.

### Updating

The `to` prop can be changed dynamically with `v-bind`. When the value changes, `<teleport>` will remove the children from the previous target and move them to the new one.

If the children contain any component instances, these will not be influenced by this. The instances will be kept alive, keep their state etc.

```html
<template>
  <teleport v-bind:to="targetName">
    <p>This can be moved around with the button below</p>
  </teleport>
  <button v-on:click="toggleTarget">Toggle</button>
  <hr />
  <div id="A"></div>
  <div id="B"></div>
</template>
<script>
  export default {
    data: () => ({
      targetName: "A"
    }),
    methods: {
      toggleTarget() {
        this.targetName = this.targetName == "A" ? "B" : "A";
      }
    }
  };
</script>
```

If the new target selector doesn't match any elements:

1. a warning should be logged during development.
2. The content would stay mounted to the previous target element.

### Destruction

When a `<teleport>` is being destroyed (e.g. because its parent component is being destroyed or because of a `v-if`), its children are removed from the DOM and any component instances destroyed just like they were still children iof the parent.

## Miscellaneous

### Naming conflict with native portals

the component introduced by this RFC was named `<portal>` in an earlier version of this RFC. But there's proposal for native portals:

- Spec: https://wicg.github.io/portals/
- Introduction: https://web.dev/hands-on-portals/

Sinc we don't want to have a naming conflict with a future HTML element that may be called `<teleport>`, especially since it's functionality is about something completely different form what portals in libs like Vue or React mean right now, we chose to rename the component to `<teleport>`

Or should we keep it as the concept of what a portal is in Vue, React e.t al. is already "common knowledge" and a new term might confuse people more than it would help?

### dev-tools

The `<teleport>` should not appear in the chain of parent components (`this.$parent`), but it should be identifiable within the virtual DOM so that Vue's dee-tools can show them in their visualisation of the component tree.

### Using a `<teleport>` on an element within a Vue app

Technically, this proposal allows to select _any_ element in the DOM , including elements that are rendered by our Vue app in some other part of the component tree.

But that puts the portal'd slot content under the control of that other component's lifecycle, which means the content can possibly be removed from the DOM if that component gets destroyed.

Any component that came through a `<teleport>` would effectively have its DOM removed by still be in the original virtual DOM tree, which would lead to patch errors when these components tried to update.

Handling this relaibly would require lots of additional logic and as such, this use case is explicitly **excluded from this RFC**. Teleporting to any DOM element that is controlled by Vue is considered to be an anti-pattern and can lead to the real dom and virtual dom being out of sync.

# Drawbacks

The only notable drawback that we see is the additional code required to implement this. But judging from experiments in the prototype, that code will be very light, as it's just a slightly different way to mount elements at the virtualDOM level, and the component itself is tree-shakable.

As it's an additive feature and the functionality is pretty straightforward (one prop defining as target selector), this should also not add much complexity to Vue in terms of documentation or teaching.

When considering how popular current userland solutions are even with their caveats and limitations, the cost/benefit ratio seems clear.

# Alternatives

No other designs were considered so far.

## What happens if we don't do this

There are currently several userland implementations of this feature available, which usually suffer some caveats and drawbacks that stem of the fact that portals are not supported at the virtual DOM level in Vue 2.

People could continue to use these with their existing limitations and drawbacks.

# Adoption strategy

`<teleport>` is a new feature and as such purely additive in nature. As such, this feature does not have any impact on the migration of Vue 2.0 applications to Vue 3.0 except for apps that might have chosen to name one of their comconents `<teleport>` - but that is easily fixable by changing that component's registration name to something else.

Users new to Vue 3.0 or Vue in general will be able to learn about this feature from the docs in the usual way and gradually introduce it into their projects where it makes sense.

## Existing 3rd party solutions

As mentioned, several 3rd party plugins/libs implement similar functionality right now.

Some of them may become irrelevant though this RFC, while others, offering functionality that exceeds what this proposal describes, would could adapt their implementation to make use of this proposal's "native" `<teleport>` component internally.

If RFC vuejs/vue-next#28 (Render function change) is adopted, these libraries will have to be reworked either way, at which point they can adopt this new feature.

# Unresolved questions

n/a
