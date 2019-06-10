- Start Date: 2019-06-10
- Target Major Version: 2.x / 3.x
- Reference Issues: N/A
- Implementation PR:

# Summary

Redesign the `$dispatch` and `$broadcast` API that existed in 1.x and deprecated in 2.x.

- `vm.$dispatch` is used to trigger specific subscription on the parent and grandparents component.
- `vm.$broadcast` is used to trigger specific subscription on the children and grandchildren component.
- Use `subscribers` to declare listeners.

# Basic example

```javascript
import Vue from 'vue'

const Item = {
  template: `<div></div>`,
  props: ['name'],
  mounted () {
    if (this.name === 'foo') {
      this.$dispatch('welcome')
    }
  },
  subscribers: {
    hello () {
      console.log(`hello ${this.name}`)
      if (this.name === 'baz') {
        return false
      }
    }
  }
}

const Root = {
  components: { Foo },
  template: `
    <div>
      <Item name="foo"></Item>
      <Item name="bar">
        <Item name="baz">
          <Item name="qux"></Item>
        </Item>
      </Item>
    </div>
  `,
  mounted () {
    this.$broadcast('hello')
  },
  subscribers: {
    welcome () {
      console.log('welcome Root')
    }
  }
}

new Vue(Root).$mount('#app')
```

Output:
```
welcome Root
hello foo
hello bar
hello baz
```

# Motivation

Although this API was deprecated, it is still very useful in many scenarios.

One of the most common examples:

```javascript
import { createEditor } from 'some-editor'

const Sidebar = {
  // Ignore the resizable implementation
  template: `
    <div>Sidebar</div>
  `,
  methods: {
    resize (width) {
      this.$el.style.width = `${width}px`
      this.$broadcast('resize')
    }
  }
}

const Content = {
  template: `
    <div>
      <slot></slot>
    </div>
  `
}

const Editor = {
  template: `
    <div ref="editor"></div>
  `,
  mounted () {
    this.editor = createEditor(this.$refs.editor)
  },
  subscribers: {
    resize () {
      this.editor.fit()
    }
  }
}

const App = {
  components: {Sidebar, Content, Editor},
  template: `
    <div>
      <Sidebar/>
      <Content>
        <Editor/>
      </Content>
    </div>
  `,
  mounted () {
    window.addEventListener('resize', () => {
      this.$broadcast('resize')
    })
  }
}
```

```
App
├─ Sidebar
└─ Content
   └─ Editor
```

The size of `App` will be affected by the size of the browser window. `Sidebar` is a resizable component that will affect the size of `Content`. `Editor` is a custom 3rd-party control (e.g. moanco-editor, xterm.js) that unable to adapt to the size of parent.

We want `Editor` can adjust its size when `App` or `Content` is resized.

For this we need to have `Editor` subscribe event `resize`. If `App` or `Sidebar` were resized, `resize` would be invoke, then `Editor` will adapt to a new size.

I have encountered a much more complicated situation than this. In particular, if I implement a custom SplitView component, it is difficult to manage `resize` events using Vuex or a global event bus.

By the way, the UI library [Element](https://github.com/ElemeFE/element/blob/f6df39e8c1ff390da0f0df8ea30b07baf5d457f0/src/mixins/emitter.js) also reimplemented `$dispatch`.

# Design Detail

Since Vue 3.0's class component and various hooks have not yet been decided, this feature can only be designed based on the API style of Vue 2.x for references.

## Subscribers
The declaration is similar to `methods`:
```javascript
export default {
    subscribers: {
      resize () {
        // ...
      }
    }
}
```

## Dispatch
```javascript
this.$dispatch('subscriber-name', 'arg1', 'arg2', '...')
```

Dispatch will bubble the event to the parent and grandparents component.
```
App
├─ Foo
└─ Bar
   └─ Baz
      └─ Qux
```
`App` `Foo` `Baz` `Qux` both have a subscriber named `hello`. If we dispatch it from `Qux`, then the subscriber `hello` in `Baz` and `App` would be trigger.

The steps is as follows:

`Qux (dispatcher will be skip)` → `Baz (trigger)` → `Bar (no matched subscriber, skip)` → `App (trigger)`

If we add a `hello` subscriber into `Bar` that return `false`, it will stop bubbling:

```javascript
export default {
    name: 'Bar',
    subscribers: {
      hello () {
        return false
      }
    }
}
```

The steps will become like this:

`Qux (dispatcher will be skip)` → `Baz (trigger)` → `Bar (return false, stop bubbling)`

## Broadcast
```javascript
this.$broadcast('subscriber-name', 'arg1', 'arg2', '...')
```

Broadcast will trigger all matched subscriber in children and grandchildren component.

 ```
 App
 ├─ Foo
 └─ Bar
    └─ Baz
       └─ Qux
 ```
All components except `Bar` have a subscriber named `hello`. If we broadcast it from `App`, then all subscribers `hello` in `Foo` `Baz` `Qux` would be trigger.


The steps is as follows:

`App (broadcaster will be skip)` → `Foo (trigger)` → `Bar (no matched subscriber, skip)` → `Baz (trigger)` → `Qux (trigger)`

The method of stop bubbling by returning `false` is also available in broadcast.

## Parameter passing
The arguments for `$broadcast` and `$dispatch` are exactly the same as `$emit`.

```typescript
export interface Vue {
  $broadcast(event: string, ...args: any[]): this
  $dispatch(event: string, ...args: any[]): this
}
```

# Drawbacks
- Add a new component option `subscribers` is a big change.
- Changes to the api will probably make it more difficult for Vue 1.x users to adapt.
- It will conflict with the existing library that has manually implemented `$broadcast` and `$dispatch`.

# Alternatives
Some self-implemented library ([Like this](https://github.com/ElemeFE/element/blob/f6df39e8c1ff390da0f0df8ea30b07baf5d457f0/src/mixins/emitter.js)).

# Adoption strategy
For libraries that have implemented similar functionality themselves, they can choose to continue using their own implementations or migrate to Vue's own implementation.

# Unresolved questions

Design Vue 3.0 / TypeScript compliant API.

- class component
- arguments-typed subscriber
- subscriber declaration combined with vue event bus
