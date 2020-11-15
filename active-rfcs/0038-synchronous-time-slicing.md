- Start Date: 2020-11-15
- Target Major Version: 3.x
- Reference Issues: none
- Implementation PR: soon

# Summary

Time slicing of microtask using requestanimationFrame.

# Basic example

This is an internal implementation and no API changes.

# Motivation

The [current scheduler implementation](https://github.com/vuejs/vue-next/blob/master/packages/runtime-core/src/scheduler.ts#L192) is using microtask and empty a queue synchronously. Because of the execution mechanism of microtask, it will block the browser. We call it `[jank](https://developer.mozilla.org/en-US/docs/Glossary/Jank)`

If we can slice the microtask properly, we can effectively alleviate this problem.

# Detailed design

Frankly, I found that RAF does not defer the microtask to the next tick, so slicing a microtask with the RAF has very little invasive.

1. As before, it's still synchronous.

2. We can't slice according to 16ms or single component, because the granularity is too small, many small components don't need a 16ms.

3. So we need a timeout queuing algorithm.

# Drawbacks

```js
let frame = 0
let queue = []
let deferQueue = []

const consume = function (queue, timeout) {
  let i = 0, ts = 0
  while (i < queue.length && (ts = performance.now()) < timeout) {
    queue[i++](ts)
  }
  if (i === queue.length) {
    queue.length = 0
  } else if (i !== 0) {
    queue.splice(0, i)
  }
}
const flush = function () {
  frame++
  const timeout = performance.now() + (1 << 4) * ~~(frame >> 3)
  consume(queue, timeout)
  consume(deferQueue, timeout)
  if (queue.length > 0) {
    deferQueue.push(queue)
    queue.length = 0
  }
  if (readQueue.length + queue.length + deferQueue.length > 0) {
    requestAnimationFrame(flush)
  } else {
    frame = 0
  }
}

export const queueJobs = (cb) => queue.push(cb) === 1 && Promise.resolve().then(flush)
```

# Alternatives

This seems to be the only way to synchronize time slicing.

React Fiber is asynchronous rendering, They are essentially different.

To avoid misunderstanding, I don't think it can be used as an alternative here.

# Adoption strategy

This proposal is very small intrusive, will not cause trouble to users

And it is not difficult to implement it, I can land it soon.

# Unresolved questions

None, everything is ready~