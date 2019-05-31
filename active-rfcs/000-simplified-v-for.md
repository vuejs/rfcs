- Start Date: 2019-05-23
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: 

# Summary

- Make `v-for` to behave more similar to  `for` and `Array#forEach` in 
  Javascript.
- Introduce `v-for-from` for iterating through objects and range.
- Introduce range syntax `..`.

# Basic example

```html
<!-- iterating over arrays -->
<div v-for="x of xs"></div>
<div v-for="(x, i) of xs"></div>

<!-- iterating over object keys -->
<div v-for="key in object"></div>

<!-- iterating over objects -->
<div v-for="(key, value) from object"></div>
<div v-for="(key, value, i) from object"></div>

<!-- iterating through range -->
<div v-for="i from 1..10"></div>
<div v-for="i from 1,5..10"></div>
```

# Motivation

In Vue 2.x, `v-for` uses the same keywords in Javascript, i.e. `in` and
`of`, which behaves differently from Javascript. This can be rather
confusing for:

1. For complete beginners, they might be mislead to think `value in obj`
   is legal in Javascript.
2. For Javascript developers who are new to Vue, they might not be fond
   of the fact that `v-for` uses the same keywords but behaves
   differently.

The motivation of this RFC is to remove this confusion and introduce
the keyword `from` which is foreign for Javascript for iteration over
objects and range.

# Detailed design

Implement `v-for` so that it maps directly to `for`-loop / `forEach` in 
Javascript.

The following behaviors are not consistent with Javascript usage and
should be removed/replaced with `from`.

1. Iterating over objects with `for-in` (`v-for-in-object`)
   ```html
   <div v-for="(value, key, index) in object"></div>
   ```
   In Vue 3.x, this should be done with the `from` keyword:
   ```html
   <div v-for="(key, value, index) from object"></div>
   ```
   Note that the position for `key` and `value` are swapped.
2. Iterating over array with `in` (`v-for-in-array`)
   ```html
   <div v-for="x in xs"></div>
   ```
   In Vue 3.x, this will still be valid but `x` will become the indices
   of `xs` instead of the values in `xs`. To iterate over an array, use
   `v-for-of`:
   ```html
   <!-- i is the index -->
   <div v-for="i in xs"></div>
   <!-- x is the value -->
   <div v-for="x of xs"></div>
   ```
3. Iterating over number (`v-for-in-number`)
   ```html
   <div v-for="i in 10"></div>
   ```
   In Vue 3.x, this will be replaced with `from` and the range syntax
   `..`:
   ```html
   <div v-for="i from 1..10"></div> <!-- 1 2 3 4 ... 10 -->
   ```
   You can also provide the second number in the range which denotes the
   step for the range.
   ```html
   <div v-for="i from 1,3..11"></div> <!-- 1 3 5 7 9 11 -->
   <div v-for="i from 1,5..11"></div> <!-- 1 5 10 -->
   ```

# Drawbacks

- Upgrading from Vue 2.x requires a lot of changes depending on how
  heavily `v-for-in-object` and `v-for-in-array` and `v-for-in-number`
  are used throughout the project.
- If the future versions of Javascript introduces `from` as a keyword,
  our implementation will likely become inconsistent again.

# Alternatives

We can even go further by not introducing `from` and `..`, remove the
`index` as the second argument. This will make the `v-for` exactly the
same as Javascript's `for`.

For example:
```html
<!-- iterating over arrays -->
<div v-for="x of xs"></div>
<div v-for="([x, i]) of xs.entries()"></div>

<!-- iterating over object keys -->
<div v-for="key in object"></div>
<div v-for="key of Object.keys(object)"></div>

<!-- iterating over objects -->
<div v-for="[key, value] of Object.entries(object)"></div>
<div v-for="([[key, value], i]) of Object.entries(object).entries"></div>
<div v-for="value of Object.values(object)"></div>
<div v-for="([value, i]) of Object.values(object).entries()"></div>

<!-- iterating through range -->
<div v-for="i of Array.from({ length: 10 }, (_, i) => i)"></div>
```


# Adoption strategy

To adopt to this simplified `v-for`, we will need to only deal with the three
cases mentioned.

1. Iterating over objects with `for-in` (`v-for-in-object`)
```html
<div v-for="(value, key, index) in object"></div>
// This will become
<div v-for="(key, value, index) from object"></div>
```

2. Iterating over array with `in` (`v-for-in-array`)
```html
<div v-for="x in xs"></div>
// This will become
<div v-for="x of xs"></div>
```

3. Iterating over number (`v-for-in-number`)
```html
<div v-for="i in 10"></div>
// This will become
<div v-for="i from 1..10"></div>
```
