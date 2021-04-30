- Start Date: 2021-04-03
- Target Major Version: 2.7 / ^3.2
- Reference Issues: https://github.com/vuejs/vue-next/issues/3102, https://github.com/vuejs/vue-next/issues/3373, https://github.com/vuejs/rfcs/discussions/276
- Implementation PR: None

# Summary

Introduce a Composition Class Component to allow better typed components.

This will allow components to have types to be inferred at runtime.

The class will be very limited and only support Vue Composition-API **like** configuration.

`Props` should be handled by Typescript, meaning that all the attributes passed to the component will be exposed as props. https://github.com/vuejs/rfcs/pull/281

# Basic example

```ts
// Simple implementation
abstract class VueComponent<Props, Emits = {}, Slots = {}, RawBindings = {}> {
  protected context = {} as SetupContext<Emits>; // passed at runtime
  protected props = {} as Props; // passed at runtime

  abstract setup(): RenderFunction | RawBindings;
}

class Comp<Item> extends VueComponent<{
  item: Item;
  id: keyof Item;
}> {
  setup() {
    // access context
    this.context;

    // access props
    this.props;

    return {
      a: ref(1),
    };
  }

  template = `<div>{{ a }}</div>`;
}
```

# Motivation

Motivation is to allow users to express complex types and infer certain prop type at "runtime" (when declaring the `<my-component />`)

```ts
// currently is using a class because of a limitation on typescript
class MyClass<T extends Record<string, any>> extends VueComponent<
  { items: T; keyPath: keyof T; cols: Array<keyof T> },

  // emits
  {
    select(item: T): void;
    "update:modelValue"(item: T): void;
    cellClicked(col: keyof T, item: T);
  },

  // slots https://github.com/vuejs/rfcs/pull/192
  {
    [K: `head:${keyof T & string}`]: { a: number };
    [Y: `item:${keyof T & string}`]: { b: string };
  }
> {
  slots: = {} as any; // https://github.com/vuejs/rfcs/pull/192

  emits = {
    select(item: T) {},
    [`update:modelValue`](item: T) {},
    cellClicked(col: keyof T, item?: T) {},
  };

  setup() {
    this.props
    // declaration for testing
    const a = this;
    const cols: Array<keyof T & string> = [];
    const slots: Record<keyof typeof a.slots, Function> = {} as any;
    const item: T = {} as any;
    // / declaration for testing

    // render example
    return ()=> h("table", [
      h("thead", [this.props.cols.map((x) => h("th", [this.context.slots[`head:${x}`](this.props.items[0])]))]),
    ]);
  }
}

// or it can be used to enhance type of `defineComponent`

class Comp<T extends Record<string, any>> extends VueComponent<
  { items: T; keyPath: keyof T; cols: Array<keyof T> },

  // emits
  {
    select(item: T): void;
    "update:modelValue"(item: T): void;
    cellClicked(col: keyof T, item: T);
  },

  // slots https://github.com/vuejs/rfcs/pull/192
  {
    [K: `head:${keyof T & string}`]: { a: number };
    [Y: `item:${keyof T & string}`]: { b: string };
  }
>

const comp = defineComponent({ /* options */}) as typeof Comp; // this will "trick" typescript and set the correct type.
```

# Detailed design

This API will is simply a way to circunvent the losing of the generics when you you extract types from a function, the class component provides the generic typing with a nice syntax (this could be done with a `function` but there's too many hacks involved)

API is kept simple and is not a replacement of `Vue Class Component`, this `Vue Class Composition` is meant to **only** support composition-api, options API are not meant to be supported

# Drawbacks

One more API to `defineComponent` and users might get confused with `Vue Class Component`

# Alternatives

There's some hacky ways to achieve this which follows some of this proposal: https://github.com/vuejs/vue-next/pull/3682

# Adoption strategy

If we implement this proposal, how will existing Vue developers adopt it? Is
this a breaking change? Can we write a codemod? Can we provide a runtime adapter library for the original API it replaces? How will this affect other projects in the Vue ecosystem?

This is intened for highly advance scenarios and not intended for begginners, the users who will take the most of this are:

- Enterprise apps, which require an outstanding level of typing.
- Micro performance improvements (https://github.com/vuejs/rfcs/discussions/276)
- Component library creators.

# Unresolved questions

- Will this confuse users? 
- Fine tunning the way users interact with the `Class Composition`