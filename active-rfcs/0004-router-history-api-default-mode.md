- Start Date: 2019-05-25
- Target Major Version: 4.x
- Implementation PR:

# Summary

Make the history mode the default setting for Vue-Router in the browser, following other framework router implementations and [Google's recommendation](https://youtu.be/PFwUbgvpdaQ?t=440).

# Basic example

```javascript
const router = new VueRouter({
  routes
  // mode should default to 'history' but currently defaults to 'hash' in the browser
})
```

# Motivation

Full disclosure: I work on Google Search.

Googlebot already supports executing JavaScript and rendering client-side apps.
Other crawlers will probably start to look at the possibility of doing that soon, too.

Currently, developers and SEOs run into issues with this: [1](https://stackoverflow.com/questions/7471523/seo-impact-using-hash-urls), [2](https://webmasters.stackexchange.com/questions/99719/does-google-index-hash-bang-urls), [3](https://support.google.com/webmasters/forum/AAAA2Jdx3sU5QyHfrwFw4M/?hl=en&gpf=%23!topic%2Fwebmasters%2F5QyHfrwFw4M)

We also [announced in 2017 to deprecate the workaround for hash URLs](https://webmasters.googleblog.com/2017/12/rendering-ajax-crawling-pages.html) and we're going through with this. The workaround proved to be causing issues for developers in many cases and debugging it wasn't always obvious.

Older browsers without support for History API would get a link to a "clean" URL (without a fragment) and would see a page refresh but on old browsers that's probably fine. It's similar to using responsive design for devices with different capabilities, IMHO.

Developers who aren't aware of the limitations of hash-based routing are getting it by default and may discover issues with their SEO only once they are deployed as testing possibilities are limited.

The upside of this switch is that people don't need to be educated about the drawbacks of hash-based URLs and don't have to change existing code or new codebases to use a non-default router mode.

# Detailed design

The Vue-Router already supports the History API, but doesn't choose it as the default.
The point of this RFC is to discuss if the default can or should be changed to History API.

# Drawbacks

Why should we *not* do this? Please consider:

- This might require additional measures to make this compatible with IE10 and older, if you want to avoid a page refresh
- I am not sure if there even is a way to polyfill / retrofit this into older browsers
- It's a breaking change as it changes default behaviour

If we continue using the hash mode as the default, people will have to find ways to mitigate the URL issue for crawlers, even if they run the JS just fine otherwise. If we switch the default, people who have to support older versions of IE may need to accept page refreshes on navigation or find workarounds.

# Alternatives

We have been suggesting to switch to history mode whenever possible or implement [dynamic rendering](https://goo.gle/dynamic-rendering) or SSR / pre-rendering with Vue.

# Adoption strategy

If vue-router 4.x will use History API by default, that's a breaking change but one that reduces changes they need to make for SEO-reasons. We should probably also make sure that the starter templates implement this properly.

# Unresolved questions

- Is there something on the implementation side that we need to retrofit to browsers that don't support History API or are we okay with page refreshes for those?
- Are there implications on other parts of Vue-Router or Vue when switching the default router mode?
