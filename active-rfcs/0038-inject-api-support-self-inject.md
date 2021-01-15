- Start Date: 2021-01-15
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/vue-next/pull/3022#issue-555432690
- Implementation PR: https://github.com/vuejs/vue-next/pull/3022

# Summary

Vue3 inject function should support self-inject.

# Basic example

```js
inject(key, defaultValue, treatDefaultAsFactory, selfInject = false);
```

Since parameter selfInject's default value is false. This change is an compatible update.

# Motivation

I just wrote a Vue3 plugin that depends on self-inject feature. The plugin could make Vue support `DI` like Angular.

In angular, we can define component scope providers. It injects from self by default. And we can use `@Skip` to inject from parent.

I wrote the plugin depends on Vue@3.0.1. It works perfectly. But when I update to Vue@3.0.2. It can not work.

Without self-inject feature, every container component must be splited into two components.
One is component self. Another is parent component and just declare providers. It is too verbose to do that.

Maybe I could use HOC to solve this problem. But correct HOC in Vue is not easy like react.

# Detailed design

Just add an extra optional parameter `selfInject` to control it inject from self or parent.

And set `selfInject=false` to keep compatible.

# Drawbacks

The inject function's complexity is increased.

# Alternatives

Maybe I could use HOC to solve this problem. But correct HOC in Vue is not easy like react.

# Adoption strategy

This is not a breaking change. No one will be affected.

# Unresolved questions

Vue@3.0.2 is a breaking change. But on one cares.
