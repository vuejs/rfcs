- Start Date: 2021-04-02
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: N/A

# Summary

- Drop IE11 support plan for Vue 3.
- Focus on backport compatible features back to Vue 2.7.

# Motivation

We have been asked about IE11 support since the start of Vue 3's development, tracing back to end of 2018. Many users have asked whether Vue 3 will support IE11, and our original plan was to release Vue 3 and let it stablizie first, and add IE11 support at a later stage. During the long development process, we've also made research and experiments for IE11 compatibility on the side, but due to the complexity involved and amount of other work at hand, it's been de-prioritized down the road.

When we take another look at the problem today in 2021, the browser and JavaScript landscape has changed quite a bit. More developers are now using modern language features, and more importantly [Microsoft itself has started to actively push users away from IE](https://techcommunity.microsoft.com/t5/windows-it-pro-blog/the-perils-of-using-internet-explorer-as-your-default-browser/ba-p/331732) with its investment in Edge. It is also [dropping IE11 support in its own major projects like Microsoft 365](https://techcommunity.microsoft.com/t5/microsoft-365-blog/microsoft-365-apps-say-farewell-to-internet-explorer-11-and/ba-p/1591666). Just a few days ago [WordPress also made the decision to drop IE11 support](https://make.wordpress.org/core/2021/03/25/discussion-summary-dropping-support-for-ie11/). IE11's global usage has [dropped below 1%](https://caniuse.com/usage-table). When we are talking about public-facing websites and apps, IE11 is on a clear fast decline.

We believe this is an opportunity to rethink IE11 support for Vue 3.

## The cost of supporting IE11 in Vue 3

### Behavior inconsistencies

Vue 2's reactivity system is based on ES5 getter/setters. Vue 3 leverages ES2015 Proxies for a more performant and complete reactivity system, which cannot be polyfilled in IE11. This is the major roadblock since it means for Vue 3 to support IE11, it essentially needs to ship two different versions with different behavior - one using the Proxy-based reactivity system, and one using ES5-getter/setter-based similar to Vue 2.

Vue 3's Proxy-based reactivity system provides near complete language feature coverage. It is able to detect many operations that are impossible or impractical to intercept in ES5, for example property addition/deletion, array indice and `length` mutations, and `in` operator checks. The same code written for the Proxy version of Vue 3 will not work in the IE11 version. Not only does this create technical complexity for us, it also creates a constant mental burden for the developers.

Our original plan was to ship both the Proxy and ES5 reactivity implementations in the development build of the IE11 version. When it is run inside a Proxy-enabled dev environment, it will detect and warn against non-IE11-compatible usage. This would work in theory, but creates an enormous amount of complexity as it requires mixing the two implementations together and risks behavior difference between development and production.

### Long-term maintenance burden

Supporting IE11 also means we have to consider language features used in the entire codebase, and figure out proper polyfill / transpilation strategy for our distribution files. Every new feature addition that cannot be polyfilled in IE11 will create yet another behavior caveat. Once Vue 3 commits to IE11 support, it won't be able to get rid of it until the next major release.

### Complexity for library authors

The complexity would still be somewhat acceptable if it can be fully contained within Vue itself. However, after discussing with community members, we realized the co-existence of two reactivity implementations inevitably leaks to library authors as well.

By supporting IE11 in Vue 3, library authors essenitally need to make that call as well. Library authors will have to account for what build of Vue 3 their library is running with (on top of potentially also supporting Vue 2) - and if they decide to support IE11, they have to author their library with all the ES5 reactivity caveats in mind.

### Contributing to IE11's staying power

Nobody enjoys supporting IE11. It is a dying browser stuck in the past. The further the web ecosystem moves forward, the larger the gap we need to cover when trying to support it. Ironically, by supporting IE11 in Vue 3 we are giving it more reasons to stick around. Given our user base, dropping IE11 support will likely help make it obsolete a bit faster.

## For those who absolutely need IE11 support

We are well aware that the real demand for IE11 comes from those who are unable to upgrade: financial institutions, education sectors, and those who rely on IE11 for screen readers. If you are building an application targeting these sectors, you may not have a choice.

Our recommendation is to use Vue 2 if you need IE11 support. Instead of incurring significant technical debt for Vue 3 and future versions of Vue, we believe it would be more meaningful to redirect the effort towards backporting compatible features to Vue 2 in the 2.7 release, and ensure a closer develpment experience across the two major versions.

Some of the features that can be backport to 2.7:

- Merge the [`@vue/composition-api` plugin](https://github.com/vuejs/composition-api) into Vue 2. This will enable Composition API based libraries to directly work for both Vue 2 and Vue 3.
- [`<script setup>`](https://github.com/vuejs/rfcs/pull/227) syntax in Single-File Components.
- `emits` option.
- Improved TypeScript typings.
- Formalize Vue 2 support in Vite (currently via [non-official plugin](https://github.com/underfin/vite-plugin-vue2))

Note: the above list is tentative/non-exhaustive and will be discussed/finalized in a separate RFC.