- Start Date: 2020-5-15
- Target Major Version: 2.x, 3.x
- Reference Issues: N/A
- Implementation PR:

# Summary
- Make it simpler to pass `v-model` down in (say) wrapper components.

# Basic Example
```html
<template>
  <label>
    {{label}}
    <input v-model="value" />
  </label>
</template>
<script>
  export default {
    model: {
      prop: 'value',
      event: 'input'
    }
    props: {
      label: String,
      value: {
        type: String,
        set(value) {
          this.$emit('input', value);
        }
      }
    }
  };
</script>
```

# Motivation
It is often required that we rewrite existing form components that accept `v-model`s in wrapper components. However, we cannot simply pass the `v-model` down from the wrapper to the wrapped component:

```html
<template>
  <label>
    {{label}}
    <!--
      What about putting a v-model here?
    -->
    <input
      :value="value"
      @input="$emit('input', $event.target.value)"
    />
  </label>
</template>
<script>
  export default {
    model: {
      prop: 'value',
      event: 'input'
    }
    props: {
      label: String,
      value: String
    }
  };
</script>
```

# Drawbacks
It may make Vue’s one-way data flow less apparent.

# Alternatives
https://github.com/vuejs/rfcs/pull/10:

```html
<template>
  <label>
    {{label}}
    <input v-model="value" />
  </label>
</template>
<script>
  export default {
    model: {
      prop: 'value',
      event: 'input'
    }
    props: {
      label: String,
      value: {
        type: String,
        as: 'valueProp'
      }
    },
    computed: {
      value: {
        get() {
          return this.valueProp;
        },
        set(value) {
          this.$emit('input', value);
        }
      }
    }
  };
</script>
```
