- Start Date: 2019-02-03
- Target Major Version: 2.x
- Reference Issues: N/A
- Implementation PR:

# Summary

Allow *re-emitting* (forwarding) an event that is not intended to be handled in the current component but we want to forward the children event behaviour to a grandparent component.

This RFC helps reducing boilerplate for a common vue.js pattern.

Introducing the new `.emit` event modifier. (Name opened to change/discussion)

# Basic example

This would be a good case where we could apply this new modifier:

```html
<!-- Children -->
<div>
  <button @click="$emit('click', $event)"></button>
</div>

<!-- Parent -->
<div>
  <children-component @click.emit />
</div>

<!-- Grand Parent -->
<div>
  <parent-component @click="handleClick" />
</div>
```

In above example, `parent-component` does not want/need to handle or add extra behaviour to _click_ event, but it wants to _re-emit_ the event upwards to allow parent components to handle it.

# More realistic example

Let's imagine we are building a custom text input. If we want to allow to react to common input events (focus, blur, etc...) we have to handle all the desired events and just _re-emit_ all the events (For the sake of the example, we won't add any custom behaviour to them):

```html
<!-- MyInput.vue -->
<div>
  <input 
    type="text"
    @focus="$emit('focus', $event)"
    @blur="$emit('blur', $event)"
    ...
    />
</div>
```

With the `.emit` modifier this could be simplified as:

```html
<!-- MyInput.vue -->
<div>
  <input 
    type="text"
    @focus.emit
    @blur.emit
    ...
    />
</div>
  
<!-- Parent --> 
<div>
  <my-input
    @focus="onFocus"
    @blur="onBlur" />
</div>
```

# Motivation

The main goal of this feature is to reduce boilerplate when building deeply nested components in which you have to emit events upwards through several component layers.

If we make this process easier, this could help to avoid less visual patterns like `provide/inject`, or a custom event bus.

# Detailed design

This RFC is meant to be _only_ syntax sugar for this pattern:

```html
<my-component @click.emit />

<!-- Transpiles to... -->

<my-component @click="$emit('click', $event)" />
```

The new modifier could be chained with the rest of the modifiers too:

```html
<!-- Without .emit -->
<form @submit.prevent="$emit('submit', $event)"> ... </form>

<!-- With .emit -->
<form @submit.prevent.emit> ... </form>
```

```html
<!-- Without .emit -->
<button @click.stop="$emit('click', $event)"> ... </button>

<!-- With .emit -->
<button @click.stop.emit> ... </button>
```

#### Templating engine
I think it doesn't make sense to have a handler with the `.emit` modifier (`@click.emit="onClick"`). Maybe a warning indicating that `.emit` modifier is meant to avoid handling the event in the current component is enough.

#### Render Function
For this case, I'd say that `.emit` doesn't need to be implemented in any way because of the nature of the render function.

I'd leave to the user to implement it as the docs say for `.stop` or `.prevent` modifiers.

# Drawbacks

##### Another modifier
It's something "new" to learn and could confuse the newcomers.

##### It's less explicit
Although I think the `.emit` modifier is a readable way of re-emitting an event, I understand that anyone could see this as a way of obfuscating what is happening.

##### The relation with the rest of modifiers
Vue has more modifiers as _key modifiers_, _mouse modifiers_, _system modifiers_, etc... In some edge cases could be confusing because of the verbosity.

```html
<input @keyup.alt.67.emit />

<!-- Translated to: -->

<input @keyup.alt.67="$emit('keyup', $event') />"
```

```html
<input @click.ctrl.exact.emit />

<!-- Translated to: -->

<input @click.ctrl.exact="$emit('keyup', $event') />"
```

# Alternatives

If this RFC is rejected, the alternative is to keep things as is right now.

# Adoption strategy

The strategy is simple. This RFC only adds a feature on top of Vue, so it only needs to be documented and explained (Similar as v-model is explained).

# Unresolved questions

##### Template handlers without value
I'm not really sure if the template engine allows to have handlers without value. I know that `@click.stop` works because I  have used it a few times, but I don't know if it's an exception. I'd need any with more expertise to answer this question.

##### Alternative names of the modifier
I think the word `emit` makes sense, especially in Vue ecosystem. It's a known word and it's a known behaviour. The word `emit` is directly related to the events, so putting it next to an event handler, makes clear what the component it's trying to do. (`@click.emit`)

I've thought in different names for the modifier that could be a better option in terms of user engagement:

1. `.forward`: This could be a better fit because of the meaning of the word. Furthermore, this word can be associated directly with proxies, which this feature tries to mimic.
2. `.propagate`: Although I like this one, I consider it a bit verbose and I think people could confuse it with the concept of `event.stopPropagation()` (Which is exactly the opposite).
