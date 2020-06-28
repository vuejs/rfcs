- Start Date: 2020-06-28
- Target Major Version: 3.x
- Reference Issues: 
- Implementation PR: 

# Summary

Refer to Scoped Slots as Slots with props from now on in docs. Refer to Slot Scope as to Slot Props.

# Motivation

Scoped Slots were introduced in 2.1 to pass arbitrary data to slot using attributes\props on a slot element
and a `slot-scope` directive on a slot consumer. In 2.6 `slot-scope` was removed in favour of a new `v-slot` directive. The only two remaining places where you can encounter the term *Scope* on a daily basis in Vue 2 are render functions (`this.$scopedSlots`) and docs. In Vue 3 `scopedSlots` do not exist in the render API anymore.

Scoped Slots is not an easy concept to grasp at first.
Partially because the term *Scope* is not explained in the documentation and is referencing slot implementation details. In particular, the render function API in Vue 2.x is using component props to render slots, so calling it `slotProps` at that time would not be convenient for developers:

```js
h('SlottedComponent', {
  // Slot Props are actually a slot argument, not the prop themselves
  slotProps: {
    default: ({ title }) => h('div', title)
  }
})
```

So Scoped Slots (as an implementation detail) was chosen instead as their name in documentation as well.

The term *Scope* is also used in the Scoped Styles, but it's not the same scope as in the Scoped Slots, which might as well be confusing for developers.

----

This RFC is focused on changing all references of Scoped Slots to Slots  with props and Slot Scope to Slot Props.

Slot Props was chosen as a new name primarily because the concept of *Props* is already familiar to Vue developers from the very beginning (they appear in Component Basics) and is easy to understand.

Scoped Slots do actually act as components with props on a high level.
The Slot component can accept props that are then passed down to a consumer via Template Render Context. These props should not be modified by developer and can be reactive (exactly as props).

Even docs themselves refer to them as to `slotProps`, this example is taken from a Scoped Slots documentation:

```html
<current-user>
  <template v-slot:default="slotProps">
    {{ slotProps.user.firstName }}
  </template>
</current-user>
```

The distinction between Slots and Scoped Slots is nonexistent in Vue 3 and having a separate name for slot props is confusing because these slots are the same except for having or not having props on them.
Without this distinction it's even harder to justify a concept of Scoped Slots in Vue 3 since it's not even render function related anymore.

This change would also leave intact current render function API in Vue 3, which is important considering the late stage of Vue 3 development.

# Detailed design

Scoped Slots should be referred to as Slots with props or Slot Props when it comes to actual slot props.

The arguments that the abstract slot component receives should always be referenced as Slot Props.

Slot Scope should also be called Slot Props.

----

In this example `foo` and `bar` are Slot Props. Previously called Scoped Slot here is now just a Slot or a Slot with props in particular.
```html
<div>
  <slot :foo="foo" :bar="bar" />
</div>
```

```html
<div>
  <slot v-bind="slotProps" />
</div>
```

Slot Scope is now also called Slot Props for convenience.
```html
<SlottedComponent v-slot="slotProps">
  {{ slotProps.foo }}
</SlottedComponent>
```

Render API left intact.
```js
h('div', [
  this.$slots.default(slotProps)
])
```

```js
h('SlottedComponent', {
  default: (slotProps) => slotProps.foo
})
```

# Drawbacks

Developers familiar with the old term would require some time to switch to the new name and newcomers might be confused with the term that is not explained in the Vue 3 documentation. (might be solved with a hint on Slot Props page with a link to Vue 2.x Scoped Slots docs)

# Adoption strategy

Scoped Slots should be replaced in Vue 3 docs prior to the final release
