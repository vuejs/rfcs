- Start Date: 2019-03-22
- Target Major Version: 2.x
- Reference Issues: https://github.com/vuejs/vue/issues/9749 https://github.com/vuejs/vue/issues/8403
- Implementation PR:

# Summary

Transparently use the `v-model` directive on data **and props**.

# Basic example

## Vue 2.6

```js
const child = {
  template: `<input type="text" :value="value" @input="$emit('input', event.target.value)">`,
  props: {
    value: String,
  },
};
const parent = {
  components: { child },
  template: `<child v-model="value">`,
  data() {
    return { value: '' };
  },
};
```

## Proposed behavior

```js
const child = {
  template: `<input type="text" v-model="value">`,
  props: {
    value: String,
  },
};
const parent = {
  components: { child },
  template: `<child v-model="value">`,
  data() {
    return { value: '' };
  },
};
```


# Motivation

Wrapping model exposing components is a rather cumbersome and error prone task,
which involves writing boilerplate code. This gets even worse when dealing with
keyed models or dynamic components (*What do I actually need to forward*).

The `v-model` improvements proposed here can also be applied to the `.sync`
modifier, depending on https://github.com/vuejs/rfcs/pull/8.

# Detailed design

While the initial implementation path seems simple, there are a lot of corner
cases to respect.

The following example should contain all corner cases that need to be respected:
```js
const editAttribute = {
  template: `
    <component
      :is="tag"
      v-bind="tag === 'input' ? { type: "text" } : {}"
      v-model="model.attributes[attribute]"
    >
  `,
  model: {
    prop: 'model',
    event: 'attributeUpdated',
  },
  props: {
    model: Object,
    attribute: String,
  },
  data() {
    return { tag: 'input' };
  }
};
const parent = {
  components: { editAttribute },
  template: `<edit-attribute v-model="list[index].data" attribute="name">`,
  props: {
    index: {
      type: Number,
      default: 0,
    },
  },
  data() {
    return {
      list: [
        {
          data: {
            attributes: {
              name: 'Foo',
            },
          },
        },
      ],
    };
  },
};
```

This should be compiled into something equivalent like this:
```js

// helper which extracts the value based on the target components type,
// basically `$event.target.value` for native elements and `arguments[0]` otherwise
function EXTRACT_VALUE(/* whatever arguments needed, to extract the value */) {
  return '';
}

const editAttribute = {
  template: `
    <component
      :is="tag"
      v-bind="tag === 'input' ? { type: "text" } : {}"
      :value="model.attributes[attribute]"
      @input="forward"
    >
  `,
  model: {
    prop: 'model',
    event: 'attributeUpdated',
  },
  props: {
    model: Object,
    attribute: String,
  },
  data() {
    return { tag: 'input' };
  },
  methods: {
    forward(event) {
      const value = EXTRACT_VALUE(/* whatever arguments needed, to extract the value */);
      const $set = INNER_FORWARD ? arguments[1] : this.$set;

      // a new 2nd argument to forwarded v-model events: "the setter thunk"
      // it is only needed, if keyed models are used
      this.$emit('attributeUpdated', value, (target, key, value) => {
        if (KEYED_MODEL) { // decided at compile time, for this example, would go into the first branch
          if (key != null) {
            $set(target[key].attributes, this.attribute, value);
          } else {
            $set(target.attributes, this.attribute, value);
          }
        } else {
          if (key != null) {
            $set(target, key, value);
          } else {
            target = value; // THIS DOES NOT WORK, parent target is unkeyed
          }
        }
      });
    },
  },
};
const parent = {
  components: { editAttribute },
  template: `
    <edit-attribute
      :model="list[index].data"
      @attributeUpdated="attributeUpdated"
      attribute="name"
    >
  `,
  props: {
    index: {
      type: Number,
      default: 0,
    },
  },
  data() {
    return {
      list: [
        {
          data: {
            attributes: {
              name: 'Foo',
            },
          },
        },
      ],
    };
  },
  methods: {
    attributeUpdated(value, setter) {
      const $set = setter || this.$set; // BC, in case the child is not compiled with this feature
      $set(this.list[this.index], 'data', value);
    },
  },
};
```

# Drawbacks

- Increased complexity in `v-model`'s implementation

# Alternatives

In https://github.com/vuejs/vue/issues/8403 a separate modifier for the
`v-model` directive was coined to enable the proposed behavior, but it is
actually not needed.

# Adoption strategy

The adoption can be done seemless as none of the *supported* existing behavior
is changed. Existing applications can mix and match between the new transparent
`v-model` behavior and manual forwarding.

# Unresolved questions

- How to do EXTRACT_VALUE?
- How to do INNER_FORWARD detection?
- BC when only parent is compiled with this feature, but child is not?
- BC when only child is compiled with this feature, but parent is not?
