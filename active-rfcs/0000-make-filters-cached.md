- Start Date: 2019-01-21
- Target Major Version: 2.x & 3.x
- Reference Issues: https://github.com/vuejs/vue/issues/8880, https://github.com/vuejs/vue/issues/8497, https://github.com/vuejs/vue/issues/7556, https://github.com/vuejs/vue/issues/5682
- Implementation PR: (leave this empty)

# Summary

Make filters work like computed properties - only run them when inputs change.

# Motivation

Copied from https://github.com/vuejs/vue/issues/8880:

There are a lot of issues created about filters: https://github.com/vuejs/vue/issues/8497, https://github.com/vuejs/vue/issues/7304, https://github.com/vuejs/vue/issues/5920, https://github.com/vuejs/vue/issues/5682, https://github.com/vuejs/vue/issues/5109 etc.  
Most of the issues are created because filters are executed multiple times.  
For all of these issues answer is the same: for vdom comparison vue needs to run filter and then compares resulting dom tree - use computed properties instead.

Because filters are evaluated on every re-render and not only when inputs change, right now there is no reason to use filters in mustache interpolation - they are the same as methods.  
Angular filters for example are only evaluated when inputs change and this makes them much more usable.
Filters are supposed to be pure functions anyway and I see no reason to not cache them (evaluate only when inputs change).

Making filters cachable opens up possibilities for using them in mustache interpolation.
Examples where cachable filters would be beneficial:
* vue-i18n by @kazupon works around this issue by having `v-t` directive to cache result of the translation. 
https://kazupon.github.io/vue-i18n/guide/directive.html#t-vs-v-t But is less flexible then mustache interpolation:
It would be more readable and flexible instead of:
```js
<p v-t="'hello'"></p>
```
to write
```js
<p>{{ "hello" | $t }}</p>
```
and it will be possible to do this:
```js
<p>{{ "hello" | $t }} {{ "world" | $t }}</p>
```
* Use Moment.js for date formatting. Moment.js is slow and because filters are not cached it is not really usable, especially when rendering lists.
* All other places for formatting text, dates etc. 
* Creating computed property for each formatted value (text, date etc) is just a lot of work and unnecessary code to write.

# Detailed design

Idea how it can be implemented in Vue: when Vue compiler encounters filter in mustache interpolation it could create computed property that calls filter method.

# Drawbacks

Why should we *not* do this?:

- not sure how to implement this functionality
- it is possible to implement memoization for filters in user space
- this will break statefull filters
- docs need to be extended to tell people to not write statefull filters
- it is already discouraged to write statefull filters and should not affect most of the applications

# Alternatives

- Alternative solution is to use different symbol (instead of pipe) to denote filters that are cachable (pure).
This way it will not be a breaking change, but this will create another syntax for people to learn.
- Just update Vue docs to clarify for developers that filters are executed on re-render.

# Adoption strategy

Making filters evaluated only when inputs change should improve performance for all apps that are using filters without need for any changes to apps.

If different symbol is used for pure filters it is easy to write codemod.

# Unresolved questions

Not sure how it can be implemented.
