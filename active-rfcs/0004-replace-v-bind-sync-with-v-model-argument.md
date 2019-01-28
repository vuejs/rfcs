- Start Date: 2019-01-28
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/vue-next/issues/8
- Implementation PR: (leave this empty)

## Summary

Deprecating `v-bind`'s `.sync` modifier and replacing it with an argument on `v-model`.

## Basic example

Instead of:

```vue
<MyComponent v-bind:title.sync="title" />
```

the syntax would be:

```vue
<MyComponent v-model:title="title" />
```

## Motivation

I've seen `v-bind.sync` cause quite a bit of confusion in Vue 2, as users expect to be able to use expressions like with v-bind (despite whatever we put in the docs). The explanation I've had the best success with is:

> Thinking about `v-bind:title.sync="title"` like a normal binding with extra behavior is really the wrong way to think about it, because two-way bindings are fundamentally different. The `.sync` modifier works essentially like v-model, which is Vue's other syntax sugar for creating a two-way binding. The main difference is that it expands to a slightly different pattern that allows you to have multiple two-way bindings on a single component, rather than being limited to just one.

Which brings me to the question: if it helps to tell users _not_ to think of `v-bind.sync` like `v-bind`, but rather to think about it like `v-model`, should it be part of the `v-model` API instead?

## Detailed design

> **NOTE**: Though not part of this proposal, `v-model`'s implementation details are likely to change in Vue 3 to make common patterns like transparent wrapper components easier to implement. When you see the `modelValue` attribute and `update:modelValue` event, know they are a placeholder for however we implement `v-model`'s special behavior with form elements and _not_ a recommendation included in this proposal.

### On an element

```vue
<input v-model="xxx">

<!-- would be shorthand for: -->

<input
  :model-value="xxx"
  @update:model-value="newValue => { xxx = newValue }"
>
```

```vue
<input v-model:aaa="xxx">

<!-- INVALID: should throw a compile time error -->
```

Note that `v-bind:aaa.sync="xxx"` does _not_ currently throw a compile time error, though it probably should since `

### On a component

```vue
<MyComponent v-model="xxx" />

<!-- would be shorthand for: -->

<MyComponent
  :model-value="xxx"
  @update:model-value="newValue => { xxx = newValue }"
/>
```

```vue
<MyComponent v-model:aaa="xxx"/>

<!-- would be shorthand for: -->

<MyComponent
  :aaa="xxx"
  @update:aaa="newValue => { xxx = newValue }"
/>
```

### Duplicating the object spread behavior of `v-bind.sync="xxx"`

The other directives with arguments are `v-bind` and `v-on`. Both of these use their argument-less versions for spreading an object, but `v-model` without an argument is already shorthand `v-model:model-value="xxx"`. I can see a few different options:

1. **Change the behavior of `v-model="xxx"` to spread an object, forcing users to write `v-model:model-value="xxx"` for the old behavior.** This would make `v-model`'s behavior more consistent with `v-bind` and `v-on`, but also create another breaking change and make the most common use case more verbose and complex.

2. **Add a new modifier (e.g. `.spread`) to `v-model`.** This would minimize breaking changes, but is inconsistent with the object spread behavior of other directives with arguments, potentially causing confusion and making the framework feel more complex overall.

3. **Detect and change the behavior for raw object values (e.g. `v-model="{ ...xxx }"`).** This would again minimize breaking changes, but is a little more consistent with the behavior of other directives with arguments, since `v-bind={ ...xxx }"` would have the same effect. I also expect this would be divisive, as some would likely find it very intuitive, while others would find it understandably confusing that using `xxx` creates radically different behavior than `{ ...xxx }`.

4. **Simply don't allow object spread with `v-model`.** This avoids the problems of the above two proposals, but has the drawback of making it more difficult for some people to migrate to Vue 3 (though probably a small minority). Templates/JSX that could benefit from the feature would also become much more tedious to write and maintain, in the best case, or unusable (forcing a refactor to a render function using `createElement`/`h`) in the worst case.

None of these are great options, but I'm probably most in favor of option 2. I'd also love to hear suggestions for other solutions I may have missed.

## Drawbacks

Beyond the inevitable pains with any breaking change, I think the pain for this syntax would be relatively minimal - partly because the `.sync` modifier is not as widely used a feature and also due to the potential easy of migrating users (see addoption strategy below).

## Adoption strategy

As a breaking change, this could only be introduced in a major version change (v3). However, I think there are a few things we could do to make migrating easier:

- Emit a warning when a `.sync` modifier is detected on `v-bind`, linking to this change's entry in the migration guide.
- Using the new migration helper, we should be able to detect and automatically fix 100% of cases where `v-bind` is used with `.sync`.

Combined, learning about and migrating even large codebases with heavy `.sync` usage should take only a few minutes.
