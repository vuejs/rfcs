- Start Date: 2020-11-18
- Target Major Version: 2.x
- Reference Issues: https://github.com/vuejs/vue/pull/11712
- Implementation PR: (leave this empty)

# Summary

Add ssr context to shouldPrefetch/shouldPreload to give more information about each request.

# Basic example

```
const renderer = createBundleRenderer(bundle, {
  template,
  clientManifest,
  shouldPreload: (file, type, context) => {
    if (file.includes('auth-page-components') && context.state.user.isLoggedIn) {
      return true
    }
  }
})
```

# Motivation

`shouldPrefetch` and `shouldPreload` gives us opportunity to remove scripts that are not needed on initial load.
Right now we can only use it for whole app, which is very limited. Best scenarios to use it is when we have conditional loading scripts.
Here are examples that will be possible with this change (and it's not possible right now):
- load scripts based on authorization,
- load locales for specific url (`de.json` for `/de`, etc),
- load scripts based on page type (for example we have big services for product and category. We know that on homepage there are no products so we don't need to load product service)


# Detailed design

This will extend current api of `shouldPrefetch` and `shouldPreload`. Right now we have file name and file type.
So what we need to do is just pass to it ssr context. Here is proposition https://github.com/vuejs/vue/pull/11712

```
  shouldPreload: (file, type, context) => {}
```

# Drawbacks

Extending api is always a drawback. Simple is better.
It should be used only as readonly, because we are going over loop and we want to make it quick.
So someone can overuse context in that place.

# Alternatives

I couldn't find any other options.

# Adoption strategy

It shouldn't affect current projects. We need only to add it and update docs.

# Unresolved questions
