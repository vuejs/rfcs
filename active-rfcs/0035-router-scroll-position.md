- Start Date: 2020-05-31
- Target Major Version: Vue Router v3 and v4
- Reference Issues: https://github.com/vuejs/vue-router/issues/3008,
- Implementation PR: (leave this empty)

# Summary

Deprecate `selector`, `x`, `y` and `offset` and introduce instead `el`, `top`, `left` (and `behavior`) that align with native apis and are more flexible.

# Basic example

```js
const router = new Router({
  scrollBehavior(to, from, savedPosition) {

    // scroll to id `can~contain-special>characters` + 200px
    return {
      el: '#can~contain-special>characters'
      // top relative offset
      top: 200
      // instead of `offset: { y: 200 }`
    }
  }
})
```

Other possible values returned by `scrollBehavior`:

```js
// scroll smoothly (when supported by the browser) to 400px from top and 20px from left
{
  top: 400,
  left: 20,
  behavior: 'smooth'
}

// scroll smoothly to selector .container
{
  el: '.container'
  behavior: 'smooth'
}

// directly pass an Element to `el`
{
  // use the fragment(to.hash) on the url but scroll to a child of it with a class `container`
  el: document.getElementById(to.hash.slice(1)).querySelector('.container')
}
```

# Motivation

## Deprecating `selector`

The existing scroll behavior accepts a `selector` property that internally uses `document.querySelector`. In vue-router@3 we currently have a workaround for selectors that match `/^#\d/` (id CSS selector starting with a number). This is because such selector is invalid, so we detect that and use `document.getElementById` instead. However, this breaks when using CSS combinators like `#1one .container` (select an element with class `.container` inside of an element with id `1one`). The truth is there ary many [others characters that need to be escaped](https://mathiasbynens.be/notes/css-escapes) and that we cannot escape everything, creating cases where the scroll behavior is harder to use. On top of that, it can be confusing for the user when vue-router throws because `document.querySelector` failed.

## Deprecating `x`, `y` and `offset`

Currently there are two ways of specifying an offset and both use an `x`/`y` coordinate system that differs from native functions: native browser functions allow the use of a [`ScrollToOptions`](https://developer.mozilla.org/en-US/docs/Web/API/ScrollToOptions) in `scrollTo` methods. This object contains `top`, `left` and `behavior` properties.

# Detailed design

Even though it is not commented anywhere in the RFC, **`scrollBehavior` can still return a Promise of any of the mentioned types**. It is omitted to focus on the shape of the return type instead of it being able to _await_ inside of `scrollBehavior`.

## Deprecating `x` and `y`

This aligns with [`Element.scrollTo`](https://developer.mozilla.org/en-US/docs/Web/API/Element/scrollTo) and makes it more natural passing extra options like [`behavior`](https://developer.mozilla.org/en-US/docs/Web/API/ScrollToOptions/behavior).

```js
{ x: 0, y: 200 }
// becomes
{ left: 0, top: 200 }
// it can accept `behavior`
{ left: 0, top: 200, behavior: 'smooth' }
```

## Deprecating `offset`

Instead of taking an `offset` option alongside `selector`, we could directly accept `x` and `y` alongside `selector` when specifying an offset:

```js
{
  selector: '#getting-started',
  y: -120,
}
```

Which, following the proposed rename, would become

```js
{
  el: '#getting-started',
  top: -120,
}
```

Omitting `el` will create an absolute offset as if we were providing `x` and/or `y` in vue-router@3.

This also aligns with `new Vue`'s `el` option

## More intuitive selectors thanks to `el`

The goal of a new `el` property, would be to provide simple, _"it just works"_, selectors for most common use cases while still allowing advanced cases:

- `to.hash` refers to an _id_ on the page and is directly provided to `el`. e.g. `el: to.hash` when `to.hash` equals `#about`, `#getting-started`, or `#symbols~work`
- `el` is provided a more generic selector, not necessarily coming from `to.hash`. e.g. `.container > main`, **anything not starting with `#`**
- `el` is provided an `Element` to allow any advanced use case e.g. when `to.hash` starts with `#` but contains a CSS selector rather than an id: `document.querySelector(to.hash)`

Thanks to allowing a raw Element, this covers all possible cases and makes it easier to provide the user with feedback when things do not work.

## Developer experience through warnings

Another important part of this change is warning in development when vue-router fails to find an element or when `document.querySelector` throws an Error. We can divide this in two cases:

### `el` starts with `#`

When `el` starts with `#`, we internally use `document.getElementById(el.slice(1))`. If it doesn't find an element, **in development** mode we try using `document.querySelector`, if we find something, we tell the user to use `el: document.querySelector(${providedEl})` and explain why. If we find nothing, show the usual warning of no element was found

### `el` doesn't start with `#`

In development mode, _try catch_ to provide an error message pointing to this great article https://mathiasbynens.be/notes/css-escapes and to [`CSS.escape`](https://developer.mozilla.org/en-US/docs/Web/API/CSS/escape) if `document.querySelector` throws. If nothing is found, display the same warning of no element was found.

# Drawbacks

- Breaking change but can be introduced through a deprecation

# Adoption strategy

- Deprecate `selector`, `x`, `y` and `offset` with a warning in v3
- Remove in v4

# Notes

- This RFC does not concern scrolling to a different element than window or scrolling multiple elements at once
