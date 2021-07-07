- Start Date: 2021-03-16
- Target Major Version: 2.x/3.x
- Reference Issues: https://github.com/vuejs/vue-next/issues/2084, https://github.com/vuejs/vue-next/issues/2077, https://github.com/vuejs/vue-next/issues/1518, https://github.com/vuejs/vue/issues/6259, https://github.com/vuejs/vue/issues/8028, https://github.com/vuejs/vue/issues/10487
- Implementation PR: https://github.com/vuejs/vue-next/pull/3414

# Summary

- Support `matchBy`, allow to specify the match target, there are two available values:
  - `'name'`: use component name to match.
  - `'key'`: use the `key` of the component to match.

- Cache management, provide a new prop called `cache`, users can provide **custom caching strategy**.

- Add two new lifecycle hooks: `onBeforeActivate` and `onBeforeDeactivate`, and the corresponding options API `beforeActivate` and `beforeDeactivate`.

# Motivation

- Provide more fine-grained control to allow users to decide which component instances need or not to be cached.
- Allow users to provide custom cache strategy implementation.
- Provides new lifecycle hooks, allowing users to customize behavior before the component is activated or deactivated, such as recording the position of the scroll bar.

Currently, we decide whether to cache the component by matching the component's name, which means that we cannot distinguish multiple instances of the same component. This usually happens when users use different keys on the same component:

```html
<KeepAlive>
  <component :is="currentTab.name" :key="currentTab.key">
</KeepAlive>
```

By specifying different keys, multiple component instances can be cached, but it raises two problems:

1. Cannot fine-grained control of which instances need to be cached and which do not
2. Cannot prun cache with `include/exclude`

For this, we introduce a new `matchBy` prop, see the detailed design below.

Although the cache limit can be set through the `max` prop, but for the current implementation, the caching strategy is still opinionated, i.e. *make the last visited latest*. This is good in most scenarios, but it can't meet the needs of users when in large SPA, especially when `KeepAlive` is used with `router-view`. So we need to allow users to customize the caching strategy:

```html
<KeepAlive :cache="customCacheInstance">
  <component :is="currentTab.name" :key="currentTab.key">
</KeepAlive>
```

# Detailed design

## matchBy

- Type: `'name' | 'key'`
- Default: `'name'`
- allow to specify the match target, there are two available values:
  - `'name'`: use component name to match.
  - `'key'`: use the `key` of the component to match.

Specify `key` as the matching target, which allows users to have fine-grained control over the same component(multi instances).


This will be cached:

```html
<KeepAlive include="foo,bar" matchBy="key">
  <Comp key="foo" />
</KeepAlive>
```

But this one won't:

```html
<KeepAlive include="foo,bar" matchBy="key">
  <Comp key="baz" />
</KeepAlive>
```

## cache

Provide custom caching strategy:

```html
<KeepAlive :cache="myCacheInstance">
  <Comp />
</KeepAlive>
```

#### The Cache Instance

A valid cache instance must have the following four methods:

```ts
export interface KeepAliveCache {
  get(key: CacheKey): VNode | void
  set(key: CacheKey, value: VNode): void
  delete(key: CacheKey): void
  forEach(
    fn: (value: VNode, key: CacheKey, map: Map<CacheKey, VNode>) => void,
    thisArg?: any
  ): void
}
```

For example:

```ts
// custom implementation
const _cache = new Map()
const cache: KeepAliveCache = {
  get(key) {
    _cache.get(key)
  },
  set(key, value) {
    _cache.set(key, value)
  },
  delete(key) {
    _cache.delete(key)
  },
  forEach(fn) {
    _cache.forEach(fn)
  }
}
```

Usage:

```html
<KeepAlive :cache="cache">
  <Comp />
</KeepAlive>
```

When users provides a custom cache, `KeepAlive` will abandon the built-in cache implementation. So the user will be responsible for:

1. Where does KeepAlive read the cache from? - Users need to implement the `get` method
2. Where does KeepAlive write the cache? - Users need to implement the `set` method
3. How to sync to the user-provided cache when KeepAlive prun the cache? - Users need to implement the `delete` method
4. KeepAlive needs to traverse the cache. - Users need to implement the `forEach` method

**These methods of caching instances are not meant to be executed manually by the user, but are used internally by the KeepAlive component**.

#### Manually prun the cache - `pruneCacheEntry`

Users can manually prun the cache, but **must** cooperate with the `pruneCacheEntry` method e.g.

```js
// custom implementation
const _cache = new Map()
const cache: KeepAliveCache = {
  get(key) {
    _cache.get(key)
  },
  set(key, value) {
    _cache.set(key, value)
  },
  delete(key) {
    _cache.delete(key)
  },
  forEach(fn) {
    _cache.forEach(fn)
  }
}

// Manually prun the cache
// the `pruneCacheEntry` must be called before deleting the cache,
// in order to unmount the instance/component correctly
cache.pruneCacheEntry(_cache.get('one')) // 1. call `pruneCacheEntry`
_cache.delete('one')                     // 2. delete the cache
```

About the `cache.pruneCacheEntry`, this method is automatically attached by the KeepAlive component when it is mounted. Its signature is as follows:

```js
export interface KeepAliveCache {
  get(key: CacheKey): VNode | void
  set(key: CacheKey, value: VNode): void
  delete(key: CacheKey): void
  forEach(
    fn: (value: VNode, key: CacheKey, map: Map<CacheKey, VNode>) => void,
    thisArg?: any
  ): void
  // here
  pruneCacheEntry?: (cached: VNode) => void
}
```

The following demonstrates how it works with [LRU cache](https://www.npmjs.com/package/lru-cache):

```js
const lru = new LRU({
  max: 10,
  // dispose: Function that is called on items when they are dropped from the cache
  dispose: function (key, value) {
    // Must call `pruneCacheEntry` in order to unmount the instance properly
    cache.pruneCacheEntry(value)
  }
})

// custom implementation
const cache = {
  get(key) {
    return lru.get(key)
  },
  set(key, value) {
    lru.set(key, value)
  },
  delete(ke) {
    lru.del(key)
  },
  forEach(cb) {
    lru.forEach(cb)
  }
}
```

#### warning for custom cache with max prop

If users provide a custom cache implementation, the KeepAlive component will not respect the `max` prop and will be warned if it is present.

## Lifecycle hooks

Similar to `beforeMount` and `beforeUnmount`, we provide corresponding `beforeActivate` and `beforeDeactivate`, also the corresponding composable APIs `onBeforeActivate` and `onBeforeDeactivate`.

# Drawbacks

- Not sure if there is a better solution for custom caching

# Adoption strategy

These are additional capabilities based on the current implementation
