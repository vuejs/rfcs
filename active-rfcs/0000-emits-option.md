- Start Date: 2019-02-27
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR:

# Summary

Make it explicit what events are emitted by the component.

# Basic example

```javascript
{

  emits: {
    submit: Object
  },
  
  data() {
    return {
      email: '',
      password: ''
    };
  },
  
  methods: {
    submit() {
      this.$emit('submit', {
        email: this.email,
        password: this.password
      });
    }
  }
  
}
```

# Motivation

Now, if the developer uses a component, he can easily check what props can be passed to the component. But unfortunatelly, he can't easily check what events he can subscribe to. With this feature, both developer and IDE will know what events are emitted by the component. Code editors will be able to add code completions when developer starts typing v-on: or @ in a component template.

# Detailed design

There should be an optional component option named `emits`.
Like props, there should be more than one allowed form:

## Array&lt;string&gt;

Array containing events names (in camel case):

```javascript
{
  emits: [
    'eventA',
    'eventB'
  }
}
```

## Object

Object with event name (in camel case) as a key and type constructor of the event argument as a value (like props):

```javascript
{
  emits: {
    eventA: Object,
    eventB: [String, Number]
  }
}
```

If you need more complex validation, you can use `validator` method:

```javascript
{
  emits: {
    eventA: {
      validator: (value) => ['value-a', 'value-b'].includes(value)
    }
  }
}
```

If the event doesn't pass any argument, the value should be null:

```javascript
{
  emits: {
    eventA: null
  }
}
```

# Drawbacks

There may be inconsistency if some developers will use `emits` option and others won't.

# Alternatives

Two things should be discussed:
- What `emits` option structure should look like?
- Should event emitting be validated (both event names and types declared in `emits` option)?

# Adoption strategy

If the event emitting isn't validated, adding the `emits` option won't require any change of the library. It should be mentioned in the official Vue Guide and API. Then code editors (e.g. Visual Studio Code, WebStorm, PhpStorm) should add code completions to the events.
