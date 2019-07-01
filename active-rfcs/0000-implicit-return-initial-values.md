- Start Date: 2019-07-01
- Target Major Version: (3.x)
- Reference Issues: See https://dev.to/stefandorresteijn/vuejs-is-dead-long-live-vuejs-1g7f
- Implementation PR: (leave this empty)

# Summary

Add implicit return of initial values.

# Motivation

Without implicit return the datas are redundancy.

```js
function useMouse() {
    // we initialized x, y values
    const x = value(0)
    const y = value(0)
    const update = e => {
        x.value = e.pageX
        y.value = e.pageY
    }
    onMounted(() => {
        window.addEventListener('mousemove', update)
    })
    onUnmounted(() => {
        window.removeEventListener('mousemove', update)
    })
    return { x, y } // we refer to { x, y }
}

const Component = {
    setup() {
        const { x, y } = useMouse() // we recupered x, y values from useMouse function
        return { x, y } // we refer to { x, y } from setup function
    },
    template: `<div>{{ x }} {{ y }}</div>`
}
```

# Detailed design

```js
function useMouse() {
    const x = value(0)
    const y = value(0)
    const update = e => {
        x.value = e.pageX
        y.value = e.pageY
    }
    onMounted(() => {
        window.addEventListener('mousemove', update)
    })
    onUnmounted(() => {
        window.removeEventListener('mousemove', update)
    })
    // implicit return of the values { x, y }
}

const Component = {
    setup() {
        const { x, y } = useMouse()
        // implicit return of the values { x, y }
    },
    template: `<div>{{ x }} {{ y }}</div>`
}
```

# Drawbacks

Maybe the return instruction is a mandatory instruction in JavaScript and this rfc cant be implemented.

# Alternatives

What is the impact of not doing this?

- more code
- more redundancy
- code more hard to read
