- Start Date: 2020-11-18
- Target Major Version: 2.x & 3.x
- Reference Issues: [233](https://github.com/vuejs/rfcs/issues/233)
- Implementation PR: N/A

# Summary

It is recommended to give `sfc template` a name prop, and support mutil template node

# Basic example

* a super component in  jsx
```jsx
// BaseCard
export default {
    methods:{
        renderContent(){
            // implements in sub class 
        }
    },
    render() {
        return (
            <div>
                {/*  header */}
                {this.renderContent()}
                {/* bottom */}
            </div>
        )
    }
}
```
* expected implements in sfc
```vue
<template name="renderContent">
    <div>
           aaa
    </div>
</template>

<script>
    export default {
        mixins: [BaseCard],
    }
</script>
```


# Motivation


# Detailed design


# Drawbacks

# Alternatives

# Adoption strategy

# Unresolved questions
