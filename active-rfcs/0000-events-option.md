- Start Date: 2019-02-27
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR:

# Summary

Make it explicit what events are emitted by the component.

# Basic example

```javascript
{

  events: {
    submit: {
      email: String,
      password: String
    }
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

There should be an optional component option named `events`.
Like props, there should be more than one allowed form:

## Array&lt;string&gt;

Array containing events names (in camel case):

```javascript
{
  events: [
    'eventA',
    'eventB'
  }
}
```

## Object

Object with event name (in camel case) as a key and type constructor of the event argument as a value (like props):

```javascript
{
  events: {
    eventA: Object,
    eventB: String
  }
}
```

It's common strategy to pass the event argument as object with many properties, so it should be also possible (and recommended) to do so:

```javascript
{
  events: {
    eventA: {
      a: Number,
      b: Number
    }
  }
}
```

If the event doesn't pass any argument, the value should be null:

```javascript
{
  events: {
    eventA: null
  }
}
```

# Drawbacks

There may be inconsistency if some developers will use `events` option and others won't.

# Alternatives

Two things should be discussed:
- What `event` option structure should look like?
- Should event emitting be validated (both event names and types declared in `events` option)?

# Adoption strategy

If the event emitting isn't validated, adding the `events` option won't require any change of the library. It should be mentioned in the official Vue Guide and API. Then code editors (e.g. Visual Studio Code, WebStorm, PhpStorm) should add code completions to the events.
