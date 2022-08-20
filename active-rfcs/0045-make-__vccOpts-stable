- Start Date: 2022-08-20
- Target Major Version: 3.x
- Reference Issues:
- Implementation PR:

# Summary

Make `__vccOpts` stable in a offical way.

# Basic example

```typescript
class MyComp {}

MyComp.__vccOpts = {data(){}}

export default MyComp
```

# Motivation

`__vccOpts` is used in vue internal code to access option definations from class constructor, but it's not stable or documented. This is a risk for community class api developers.

