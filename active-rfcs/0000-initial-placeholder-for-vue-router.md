- Start Date: 2019-08-20
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/vue-router/pull/2799
- Implementation PR: https://github.com/vuejs/vue-router/pull/2799

# Summary

The way I use `<router-view>` is that it has a placeholder using the `<router-view>` children before
rendering it with the actual view from the component. Here's a little demo:

https://codesandbox.io/s/vue-routing-example-b1t5d

You see that there's a slight delay before the route component gets rendered. It gets quite apparent when you have a `beforeEach` navigation guard that delays the route to be rendered. The 'blinking' effect
between the blank page and the rendered component is undesirable and I am trying to improve it.

I tried using [async component](https://vuejs.org/v2/guide/components-dynamic-async.html#Async-Components) to no avail, and it turned out that the cause was when the router is initialized, it is validating an [initial route](https://github.com/vuejs/vue-router/blob/v3.0.6/src/util/route.js#L52), and it is rendering an empty node since there is [no match](https://github.com/vuejs/vue-router/blob/v3.0.6/src/components/view.js#L51). This explains the blank page/element that I was facing.

My proposal is to render the children (that acts as a placeholder) instead of an empty node, so that there is something to be rendered before the route component takes place, which (for now) applies to any kinds of route mismatches. I did this RFC based on [a suggestion](https://github.com/vuejs/vue-router/pull/2799#issuecomment-519885182) from @posva so that I can get more feedback on this.

# Basic example

In the aforementioned PR, I've included an e2e test to demonstrate what I meant. This code is taken from [the PR](https://github.com/briwa/vue-router/blob/render-children-initial/examples/placeholder/app.js#L31).

```javascript
new Vue({
  router,
  template: `
    <div id="app">
      <h1>router-view placeholder</h1>
      <router-view name="header">
        <div id="header-loading" class="placeholder">Loading header...</div>
      </router-view>
      <router-view>
        <div id="default-loading" class="placeholder">Loading default...</div>
      </router-view>
      <router-view name="footer"></router-view>
    </div>
  `
}).$mount('#app')
```

The children of the `<router-view>` will be rendered first when `vue-router` has just been initialized and there's no matching route as mentione above. The moment there is a matched route, the actual content of the `<router-view>` will be rendered instead, replacing the `<router-view>` itself and the placeholder.

# Motivation

The initial "blank" content, in my opinion, is undesirable in terms of the UX perspective and I'm looking for a more seamless transition instead.

# Detailed design

The simplest implementation I could think of is to render the children of the route for whenever there are no matches in the route. This code is taken from [the aforementioned PR](https://github.com/briwa/vue-router/blob/render-children-initial/src/components/view.js#L52).

```javascript
// The component child may act as the temporary placeholder
// up until the actual component from the route takes place.
// Using the first child because the actual component can only
// have one single element as the root anyway.
if (children && children.length === 1) {
  return children[0]
}
```

Before the code was there, `null` was returned instead and it will render nothing on the browser (`<!---->`). With this code, it will render the children of `<router-view>` instead, providing a seamless transition between the actual component and the `<router-view>`.

# Drawbacks

- It will have some conflicts with how `vue-router` would handle 404 pages in the future, as mentioned by @posva in the PR. In my opinion, it can be handled separately outside of the PR that implements this,
since the context of HTTP responses might not be what this RFC is aiming for in the first place. I would suggest to make this as a default behavior (which will also apply for whenever 404s are happening), then the 404 case can be improved from there.
- There might be problems on what kind of placeholder should be allowed, but in my opinion, as long as it's a valid HTML, it should be fine. Feel free to point out if it isn't, though.

# Alternatives

- An alternative would be to combine this approach with how we handle 404 pages, if it was deemed necessary to be done in this phase. We could look into adding more context on the HTTP responses and how to handle those. We might produce a cleaner approach, but I foresee that it would be another big, different topic to tackle,
as opposed to this one.

# Adoption strategy

We could communicate this changes through docs and the release notes, as usual.

# Unresolved questions

- I haven't looked into the impacts of implementing this on Vue Devtools and how it would look like. AFAIK it should be ok, but if anyone knows about this, let me know.
- Also I haven't looked into how this would look like when there are `<transition>`s in the `<router-view>` itself, but AFAIK it should be ok.
