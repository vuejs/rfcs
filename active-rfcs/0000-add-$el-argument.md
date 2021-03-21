- Start Date: 2019-04-29
- Implementation PR: https://github.com/vuejs/vue/pull/11967

# Summary
- Add `$el` as a argument `($el)` inside event handler.

# Basic example

```vue
<div @click="handler($el)"></div>
```
# Motivation

Currently in vue we don't have direct access to the element upon which event is attached, Currently we can achive this by using `ref` or `$event`. Here `$event` is not reliable as `$event.target` returns actual target element not the original element upon which event is attached.

The idea behind this RFC is to make original event context available inside the `events` and it's very convenient in terms of its usage.

# Adoption strategy

It's partial alternative for `ref`'s usability, I don't think it require Adoption strategy.

# Links
[Discussion thread](https://github.com/vuejs/rfcs/discussions/288)