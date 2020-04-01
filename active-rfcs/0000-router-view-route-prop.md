- Start Date: 2020-04-01
- Target Major Version: Vue Router v3 and v4
- Reference Issues:
- Implementation PR:

# Summary

Add a `route` prop that allows customizing what is displayed by the `router-view` component. This allows users to display a view that is not the one connected to the URL. It enables patterns like modals (https://github.com/vuejs/vue-router/issues/703#issuecomment-428123334)

# Basic example

Imagine a `backgroundView` variable that allows us to know if we are displaying a different route. We could then compute a route location that would resolve to a different location than the one displayed on the url:

```js
const App = {
  computed: {
    routeWithModal() {
      if (backgroundView) {
        return this.$router.resolve(backgroundView)
      } else {
        return this.$route
      }
    }
  }
}
```

We could pass that variable to `router-view`:

```vue
<router-view :route="routeWithModal"></router-view>
```

This will in most scenarios display the component associated with the current url (`this.$route`), and in others display something different while still exposing the resolved location at `this.$route`.

# Motivation

By design, Vue Router associates a URL with a Component. This is sometimes a limitation, like with modals, and this allows escaping the limitation. It's very hard to implement a userland solution without hacks (like modifying `this.$route` at [this example](https://github.com/tmiame/vue-router-twitter-style-modals)) because it involves changing the current location. Exposing a way to modify what is displayed by `router-view` is straightforward from an implementation point of view.

# Detailed design

Vue Router already exposes the mechanism to resolve a location so users can use some global state (ideally it should be held in `window.history.state` so it is restored when navigating using the browser back and forward buttons) to generate a resolved location that can be consumed by `router-view`.

Internally, `router-view` must make sure its children (nested views) use that same location to be consistent. In v4, using `inject`/`provide` should make this simple, in v3, we might need to look up like we do for _depth_. If a `route` prop is provided, it takes precedence against any _providen_ route on the `router-view` receiving the prop **and all its children**.

The end user API is one single prop:

```vue
<router-view :route="routeWithModal"></router-view>
```

# Drawbacks

- For this mechanism to work with _Lazy Loading_, it requires for the resolved route to have been navigated to before. This ensures any lazy view has been already fetched and catched. In the scenario of modals, this is automatic because we are already on the view we want to make `router-view` display. I think this limitation makes sense because I cannot think of other use cases apart from _Modals_ when it comes to displaying a view that is not associated to the URL.

# Alternatives

- A composition function `useView` that returns a reactive component to be used with `<component :is="resultOfUseView"/>`. This solutions is way more complicated than a prop.
