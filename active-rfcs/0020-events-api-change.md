- Start Date: 2020-01-21
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/vue/issues/5443 https://github.com/vuejs/vue-next/issues/635
- Implementation PR: N/A

# Summary

Remove `$on`, `$off` and `$once` instance methods. Vue instances no longer implement the event emitter interface.

# Motivation

Vue 1.x implemented the component event system similar to that of AngularJS, with `$dispatch` and `$broadcast` where components in a tree can communicate by sending events up and down the tree.

In Vue 2, we removed `$dispatch` and `$broadcast` in favor of a more state-driven data flow (props down, events up).

With Vue 2's API, `$emit` can be used to trigger event handlers declaratively attached by a parent component (in templates or render functions), but can also be used to trigger handlers attached imperatively via the event emitter API (`$on`, `$off` and `$once`). This is in fact an overload: the full event emitter API isn't a part of the typical inter-component data-flow. They are rarely used, and there are really no strong reason for them to be exposed via component instances. This RFC therefore proposes to remove the `$on`, `$off` and `$once` instance methods.

# Adoption strategy

In Vue 2, the most common usage of the event emitter API is using an empty Vue instance as an event hub. This can be easily be achieved by using an external library implementing the event emitter interface, for example [mitt](https://github.com/developit/mitt).

These methods can also be easily provided in compatibility builds.
