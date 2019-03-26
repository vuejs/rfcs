- Start Date: 03-05-2019
- Target Major Version: 2.x & 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

Introduce APIs for dynamically injecting component lifecycle hooks.

# Basic example

```js
import { onMounted, onUnmounted } from 'vue'

export default {
  beforeCreate() {
    onMounted(() => {
      console.log('mounted')
    })

    onUnmounted(() => {
      console.log('unmounted')
    })
  }
}
```

# Motivation

In advanced use cases, we sometimes need to dynamically hook into a component's lifecycle events after the component instance has been created. In Vue 2.x there is an undocumented API via custom events:

``` js
export default {
  created() {
    this.$on('hook:mounted', () => {
      console.log('mounted!')
    })
  }
}
```

This API has some drawbacks because it relies on the event emitter API with string event names and a reference of the target component instance:

1. Event emitter APIs with string event names are prone to typos and is hard to notice when a typo is made because it fails silently.

2. If we were to extract complex logic into external functions, the target instance has to be passed to it via an argument. This can get cumbersome when there are additional arguments, and when trying to further split the function into smaller functions. When called inside a component's `data()` or lifecycle hooks, the target instance can already be inferred by the framework, so ideally the instance reference should be made optional.

This proposal addresses both problems.

# Detailed design

For each existing lifecycle hook (except `beforeCreate`), there will be an equivalent `onXXX` API:

``` js
import { onMounted, onUpdated, onDestroyed } from 'vue'

export default {
  created() {
    onMounted(() => {
      console.log('mounted')
    })

    onUpdated(() => {
      console.log('updated')
    })

    onDestroyed(() => {
      console.log('destroyed')
    })
  }
}
```

When called inside a component's `data()` or lifecycle hooks, the current instance is automatically inferred. The instance is also passed into the callback as the argument:

``` js
onMounted(instance => {
  console.log(instance.$options.name)
})
```

### Explicit Target Instance

When used outside lifecycle hooks, the target instance can be explicitly passed in via the second argument:

``` js
onMounted(() => { /* ... */ }, targetInstance)
```

If the target instance cannot be inferred and no explicit target instance is passed, an error will be thrown.

### Injection Removal

`onXXX` calls return a removal function that removes the injected hook:

``` js
// an updated hook that fires only once
const remove = onUpdated(() => {
  remove()
})
```

# Appendix: More Examples

Pre-requisite: please read the [Advanced Reactivity API RFC](https://github.com/vuejs/rfcs/pull/22) first.

When combined with the ability to create and observe state via standalone APIs, it's possible to encapsulate arbitrarily complex logic in an external function, (with capabilities similar to React hooks):

## Data Fetching

This is the equivalent of what we can currently achieve via scoped slots with libs like [vue-promised](https://github.com/posva/vue-promised). This is just an example showing that this new set of API is capable of achieving similar results.

``` js
import { value, computed, watch } from 'vue'

function useFetch(endpointRef) {
  const res = value({
    status: 'pending',
    data: null,
    error: null
  })

  // watch can directly take a computed ref
  watch(endpointRef, endpoint => {
    let aborted = false
    fetch(endpoint)
      .then(res => res.json())
      .then(data => {
        if (aborted) {
          return
        }
        res.value = {
          status: 'success',
          data,
          error: null
        }
      }).catch(error => {
        if (aborted) {
          return
        }
        res.value = {
          status: 'error',
          data: null,
          error
        }
      })
    return () => {
      aborted = true
    }
  })

  return res
}

// usage
const App = {
  props: ['id'],
  data() {
    return {
      postData: useFetch(computed(() => `/api/posts/${this.id}`))
    }
  },
  template: `
    <div>
      <div v-if="postData.status === 'pending'">
        Loading...
      </div>
      <div v-else-if="postData.status === 'success'">
        {{ postData.data }}
      </div>
      <div v-else>
        {{ postData.error }}
      </div>
    </div>
  `
}
```

## Mouse Position

``` js
import { value, onMounted, onDestroyed } from 'vue'

function useMousePosition() {
  const x = value(0)
  const y = value(0)

  const onMouseMove = e => {
    x.value = e.pageX
    y.value = e.pageY
  }

  onMounted(() => {
    window.addEventListener('mousemove', onMouseMove)
  })

  onDestroyed(() => {
    window.removeEventListener('mousemove', onMouseMove)
  })

  return { x, y }
}

export default {
  data() {
    const { x, y } = useMousePosition()
    return {
      x,
      y,
      // ... other data
    }
  }
}
```
