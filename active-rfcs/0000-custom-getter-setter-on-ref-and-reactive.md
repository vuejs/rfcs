- Start Date: 2020-01-22
- Target Major Version: 3.x
- Implementation PR: (leave this empty)

# Summary

Currently, it is possible to define getter and setter in computed.
There is no such possibility for ref / reactive. However, this functionality may be useful.

# Motivation

Implementation will allow you to create more flexible entities without the need for additional calculated values.  

# Detailed design

If the programmer does not specify getter and setter, then the behavior is identical to the current one.

```javascript
import { ref, reactive } from 'vue'

let primitive = ref(0, {
    get: function (value) {
        return value * 2;
    },
    set: function (value) {
        if (value < 0) {
            value = 0;
        }

        return value;
    }
})

let obj = reactive(
    {
        tracked: 0,
        untracked: 0
    }, 
    {
        get: function (target, property, value) {
            if (property === 'tracked') {
                return value * 2;
            }
            
            return value;
        },
        set: function (target, property, value) {
            if (property === 'tracked' && value < 0) {
                return 0;
            }
            
            return value;
        }
    }
)

// usage for ref
primitive.value = -1;
console.log(primitive.value); // must be 0

primitive.value = 10;
console.log(primitive.value); // must be 20

// usage for reactive
obj.tracked = -1;
console.log(obj.tracked); // must be 0

obj.tracked = 10;
console.log(obj.tracked); // must be 20

obj.untracked = -1;
console.log(obj.untracked); // must be -1

```

reactive can also support other methods similar to Proxy (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy)

For example: 

```javascript
import { reactive } from 'vue'

let obj = reactive(
    {
        dynamicFields: 0 
    }, 
    {
        deleteProperty: function (target, property) {
            target.dynamicFields--;
        },
        defineProperty: function (target, property) {
            target.dynamicFields++;
        }
    }
)

```

# Drawbacks

Why should we *not* do this:

- habit of using as before

# Alternatives

Continuing to use computed which leads to more confusing code. Increase memory usage. 

# Adoption strategy

Fully compatible with previous behavior. 
