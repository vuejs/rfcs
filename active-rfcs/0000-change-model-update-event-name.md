- Start Date: 2020-08-30
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

Change model update event name from `update:[modelName]Value` to `modelUpdate:[modelName]`.

# Basic example

Before the change:

```html
<FooComponent
  :modelValue="modelValue"
  @update:modelValue="onModelUpdate"
/>
```

After the change:

```html
<FooComponent
  :modelValue="modelValue"
  @modelUpdate="onModelUpdate"
/>
```

# Motivation

`update:[name]` event naming scheme for model update events follows a pattern: for every prop that could change add an `update:` prefix.

In theory this could work for values outside of model, for example:

```html
<FooComponent :value="value" @update:value="onValueUpdate" />
```

But it doesn't exactly fit Vue's data flow paradigm where we already have a `v-model` directive for two-way binding. An example above would create a one-way biding that tries to act as a two-way binding and you can't use `v-model` in that case.

This also affects events that are not related to `v-model`, but reflect some internal state:

```html
<FooComponent @update:status="onStatusUpdate" />
```

In this example `status` is an internal property but the event looks like a `v-model` event, even though it's missing a `Value` postfix.

So in general having an abstract `update:` prefix would only make sense if could apply a `v-model` on a component.

Model update event also goes into conflict with the new `emits` option that describes event listeners that should not fallthrough. In specific, the fallthrough mechanism must filter all events that start with `update:` for model event listeners to work properly with the attribute inheritance. It means that `update:` now is a reserved part of event name, but at the same time its name doesn't reflect that it is model bound.

Lastly, it is easier to understand why the event is called `modelUpdate` or `modelUpdate:foo` rather than an `update:modelValue` or `update:fooValue` if you're learning Vue. There's a direct relation to model in the event name itself, while in the case of `update:fooValue` there's no such relation.

# Detailed design

* `update:modelValue` should become `modelUpdate`

  ```html
    <FooComponent
      :modelValue="modelValue"
      @modelUpdate="onModelUpdate"
    />
  ```

* `update:fooValue` should become `modelUpdate:foo`, where `foo` is a model argument (`v-model:foo`)

  ```html
    <FooComponent
      :modelValue:foo="modelValueFoo"
      @modelUpdate:foo="onModelUpdateFoo"
    />
  ```

# Drawbacks

Due to the very late stage of Vue development this change would require significant effort to implement at the moment.

While keeping that in mind this change is beneficial in a long run when Vue 3 gains substantial adoption and developers would need to form best practices. With `update:modelValue` it would be required for `update:` events to be used only on models. With the suggested change this practice would be already in place since you can't mix up `modelUpdate` event name with a regular event name.

# Adoption strategy

Docs for Vue 3 would have to be updated to reflect these changes.

Migration from Vue 2 would be easier:

* `input` update would become `modelUpdate`
* `update:foo` for `.sync` props would become `modelUpdate:foo`