- Start Date: 2020.02.01
- Target Major Version: VTU beta-0.3x/1.x
- Reference Issues:
- Implementation PR:

# Summary

Allows the Vue Test Utils APIs that trigger re-renders to be awaited. This makes asserting changes after re-renders easier.

# Basic example
```js
    const wrapper = mount(Component)
    await wrapper.find('.element').trigger('click')
    expect(wrapper.emitted('event')).toBeTruthy()
```
# Motivation

With the removal of `sync` mode in Vue Test Utils `beta-29`, we now need to `await` for things like watchers or the template to re-render, before we make assertions.

This proposal aims to make it easier for developers to work with async tests, by allowing you to  `await` calls like `trigger` or those that use it internally. 

The proposal assumes the usage of `async/await` inside tests.

# Detailed design

The general idea is to return a promise resolving on `nextTick`, from actions that trigger async changes. 

At this moment we need to manually `await` for the component to update. This leads to extra boilerplate, and is harder to grasp for beginners.

```js
    const wrapper = mount(Component)
    wrapper.find('.element').trigger('click')
    // we need to wait for the component to render
    await wrapper.vm.$nextTick()
    expect(wrapper.emitted('event')).toBeTruthy()
```

The new API should look like:

```js
    const wrapper = mount(Component)
    await wrapper.find('.element').trigger('click')
    expect(wrapper.emitted('event')).toBeTruthy()
```

With more complicated tests, the benefits are obvious:

```js
    await wrapper.find('.button').trigger('click')
    expect(wrapper.emitted('event')).toBeTruthy()
    
    await wrapper.find('.country-option').setValue('US')
    expect(wrapper.find('.state').exists()).toBe(true)
    
    await wrapper.find('.radio-option').setChecked()
    expect(wrapper.find('.finish').attributes('disabled')).toBeFalsy()  
```

**Methods that should return a promise, resolving on next tick:**

- trigger
- setValue
- setChecked
- setData
- setProps
- setSelected
- setValue

Most of the above helpers rely on trigger internally, so updating the majority of listed methods would be easy.

### Additional helpers

Currently we have seen 3 ways to await for changes:

```js
    import flushPromises from 'flush-promises'
    await flushPromises()
    // or the more preferred
    await wrapper.vm.$nextTick()
    await Vue.nextTick()
```

In Vue 3 `nextTick` will be removed from the VM instance, so users will have to migrate over to importing it from Vue directly `import { nextTick } from 'vue'`. 

A `tick` helper can be added to the VTU exports. That way users have everything in one place, and an official way to await for renders, from actions that cannot directly return a promise.

Such a case is triggering a custom Vue event.

```js
    import { mount, tick } from '@vue/test-utils'
    
    it('test' => {
    	const wrapper = mount(Component)
     	wrapper.find('.input').vm.$emit('input', 'Newly added note')	
    	await tick()
    
    	expect(wrapper).toMatchSnapshot()
    })
```

Or we could be focusing an element on mounted, which is usually done on next tick

```js
    const wrapper = mount(Component)
    // await data to be focused on mounted
    await tick()
    let input = wrapper.find('input').element
    expect(input).toBe(document.activeElement)
```

# Drawbacks

Each `trigger` will now call `nextTick()` , which I am not sure if it can hurt performance.

Users may try to `await` anything that interacts with the DOM, so it is a matter of writing good DOCs and guides on the topic.

# Alternatives

# Adoption strategy

### Vue Test Utils beta-30+

Users would have to make sure `async/await` is working in their testing env.

Users just have to remove extra `$nextTick` and `flushPromises` calls and use the new api.

Improve docs on the topic.

### Vue Test Utils prior to beta-30

Before the removal of `sync` mode it was not necessary to await for renders.

# Unresolved questions
