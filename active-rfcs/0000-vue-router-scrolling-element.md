- Start Date: 2019-05-22
- Target Major Version: Both?
- Reference Issues: https://github.com/vuejs/vue-router/issues/1187
- Implementation PR: https://github.com/vuejs/vue-router/pull/2780 (already did the PR so...) 

# Summary

### Vue Router - Scroll Behavior 

In a lot of cases, the scrolling element in an app is not the window, 
this RFC attempts to allow the user to specify the scrolling element in their app.

# Basic example

This should be pretty straight forward, in the router options:

```javascript
new Router({
   //...
   scrollElement: '.scroller' //string or Dom Reference
})
```

# Motivation

In order to allow for more cases of scrolling, and saving positions of scrolling 
and scrolling with scrollBehavior when the window is not the scroller, this was requested
a while ago already in https://github.com/vuejs/vue-router/issues/1187. 

This shouldn't incur any breaking changes, only adding an extra option, so should be seamless to the users.

# Detailed design

Element Selector is a new property on the router instance (options),
and can also be passed by scrollBehavior results (same as selector is passed) to override the global one.

The scrolling itself is pretty simple - you get the scrolling element by passing the selector/dom element/null:

```javascript
function getScrollElement (scrollElementSelector?: any): Element | WindowProxy {
  let domScrollElement = window
  if (!scrollElementSelector) {
    return window
  }

  assert(typeof scrollElementSelector === 'string' || scrollElementSelector instanceof Element, 'Scroll Element must be a css selector string or a DOM element')
  if (typeof scrollElementSelector === 'string') {
    const customScrollElement = document.querySelector(scrollElementSelector)
    if (customScrollElement) {
      domScrollElement = customScrollElement
    }
  } else if (scrollElementSelector instanceof Element) {
    return scrollElementSelector
  }

  return domScrollElement
}
```

Using this element you can save, and / or scroll.


# Drawbacks

- Adds a bit of complexity to saving and setting scroll - need to access different properties (scrollTo  / scrollLeft / pageXOffset etc) because of old browsers quirks.
- Need to pass more info between functions that were purer before (such as saveScrollPosition that now needs to get an optional scroll selector)
- If scroller changes this might cause some unexpected behaviors.


Why should we *not* do this? Please consider:

- Slight increase in complexity
- New slightly complicated tests scenarios.
- Will need to write twice (v2 and v3)

# Alternatives

Another way that was discussed in the the original issue is to allow the user to define 2 new functions - saveScrollPosition and scrollTo
which the router will use instead of the default ones.

Doing it this way (element selector) will solve for most of the cases in a simpler setup.

# Adoption strategy

Incremental Adoption - users don't need to actively change anything if upgrading.

# Unresolved questions

The save scroll position required sending the scrolling element from a lot of different places - perhaps there is a better way to do this.
If the scroll element changes it may cause unexpected behavior with saved positions - but I am not sure it's worth worrying about - since it's only on "pop" states where it shouldn't really change.
