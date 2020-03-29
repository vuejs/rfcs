# API Deprecation [RFC]

Created: Mar 28, 2020 7:39 PM

- Start Date: 2020.03.28
- Target Major Version: 1.x
- Reference Issues:
- Implementation PR:

# Summary

Deprecate lesser used or bad practice methods and properties from the upcoming VueTestUtils 1.x API, aimed at Vue 3. 

## Motivation

With the current version of Vue Test Utils-beta.x, we notice the API is large and somewhat confusing to most  users, which are not well acquainted with testing Vue apps, leading to bad testing practices. 

This PR aims to trim down the VTU API, replacing listed methods with improved docs, more examples for all the related cases or improved testing methods.

## Detailed design

### Methods

**Wrapper.emittedByOrder -** [Link](https://vue-test-utils.vuejs.org/api/wrapper/#emittedbyorder)

Rarely used, use `emitted` instead.

    expect(wrapper.emitted('change')[0]).toEqual(['param1', 'param2'])

**get -** [Link](https://vue-test-utils.vuejs.org/api/wrapper/#get) 

Recently added, throws error if nothing is matched. Will be merged into `find`.

    expect(wrapper.find('.notExisting')).toThrow()
    
    const element = wrapper.find('.notExisting') // will throw error

**is -** [Link](https://vue-test-utils.vuejs.org/api/wrapper/#is)

Not that much useful. Use `element.tagName` instead. 

    expect(wrapper.element.tagName).toEqual('div')

**isEmpty -** [Link](https://vue-test-utils.vuejs.org/api/wrapper/#isempty)

Use custom matcher like [jest-dom](https://github.com/testing-library/jest-dom#tobeempty) on the element

    expect(wrapper.element).toBeEmpty()

**isVisible** - [Link](https://vue-test-utils.vuejs.org/api/wrapper/#isvisible)

Use custom matcher like [jest-dom](https://github.com/testing-library/jest-dom#tobevisible)

    expect(wrapper.element).toBeVisible()

**isVueInstance -** [Link](https://vue-test-utils.vuejs.org/api/wrapper/#isvueinstance)

Rarely used, no benefits.

**props -** [Link](https://vue-test-utils.vuejs.org/api/wrapper/#props)

Anti-pattern. Test what a prop does, not it's presence or value on the wrapper.

**setData -** [Link](https://vue-test-utils.vuejs.org/api/wrapper/#setdata)

Anti-pattern. Use `data` mounting option to set a specific state.

    const wrapper = mount(Component, { 
    	data () { 
    		return { 
    			field: 'overriden'  
    		}  
    	} 
    })
    

If you really need to update something after it mounts, just use the instance

    wrapper.vm.field = 'updated field'

**setMethods** - [Link](https://vue-test-utils.vuejs.org/api/wrapper/#setmethods)

Anti-pattern. Vue does not support arbitrarily replacement of methods, nor should VTU.

If you need to stub out an action, extract the hard parts away. Then you can unit test them as well.

    // Component.vue
    import { asyncAction } from 'actions'
    const Component = {
    	...,
    	methods: {
    		async someAsyncMethod() {
    			this.result = await asyncAction()
    		}
    	}	
    }
    
    // spec.js
    import { asyncAction } from 'actions'
    jest.mock('actions')
    asyncAction.mockResolvedValue({ foo: 'bar' })
    
    // rest of your test

**setProps -** [Link](https://vue-test-utils.vuejs.org/api/wrapper/#setprops)

Overriding props after the component is mounted is a hack, which introduced lots of errors, especially with watchers in VTU Beta. 

If you need to change a prop, set it on mounted.

    mount(Component, {
    	props: {
    		propA: 'value'
    	}
    })

If you need to test a prop watcher, wrap your component and test like that.

    import { h } from 'vue'
    
    const Parent = {
    	data: () => ({ propA: 'A' })
    	render() { return h(Component, { propA: this.propA }) }
    }
    
    const wrapper = mount(Parent)
    
    wrapper.vm.propA = 'B'
    // assert

**text** - [Link](https://vue-test-utils.vuejs.org/api/wrapper/#text)

Use the native element to assert text content.

    expect(wrapper.element.text()).toContain('anything')

### Mounting Options

**context** - [Link](https://vue-test-utils.vuejs.org/api/options.html#context)

Should not be needed any more.

**scopedSlots** - [Link](https://vue-test-utils.vuejs.org/api/options.html#scopedslots) 

merged with **slots** now.  

    mount(Component, {
    	slots: {
    		normalSlot: 'text',
    		scopedSlot: '<div>{{ props.slot }}</div>',
    		scopedSlot2: () => h('div', this.bar)		
    	}
    })

## Drawbacks

- Deprecations will require people to go and adjust their tests , which could make migration harder.

## Alternatives

- depending on popular demand we may leave/re-add some methods if possible.

## Adoption Strategy

- Vastly improved documentation, examples and recipes for many of the common cases.
