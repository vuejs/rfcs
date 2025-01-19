- Start Date: (2025-01-19)
- Target Major Version: (3.x)
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

Expand the warning information of validator verification to provide clearer error prompts.

# Basic example

```vue
<script setup>
defineProps({
  num: {
    type: Number,
    extendValidator(name, value, props, warn) {
			if (typeof value !== 'number') {
				warn(`[Custom Component]: Invalid prop: custom validator check failed for prop "${name}", expected type number, but got ${typeof value}`)
			} else (!Number.isInteger(value)) {
				warn(`[Custom Component]: Invalid prop: custom validator check failed for prop "${name}", expected an integer.`)
			}
    }
  }
})
</script>
```

# Motivation

In custom components, we usually validate many props for a better user experience. Currently, the validator function will only return true or false. If it is false, it means that the validation has failed and a warning message will be thrown inside vue.

However, the warning message only indicates that a prop check failed, but users cannot customize and expand more specific error descriptions.

# Detailed design

In order not to affect the original users, it may be a good idea to add a new function to give users more customization space.

refer #Basic example

# Drawbacks

In most cases, the current validator function will suffice. (This proposal is merely to optimize the development experience based on the original one.)

# Alternatives

Add new parameters based on the original validator and make different processing according to different parameters.

# Adoption strategy


# Unresolved questions

N/A