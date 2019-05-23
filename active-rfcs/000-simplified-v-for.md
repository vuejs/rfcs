- Start Date: 2019-05-23
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: 

# Summary

- Make `v-for` to behave more similar to  `for` and `Array#forEach` in 
  Javascript.
- Remove the "magic" in `v-for`.

# Basic example

```html
<!-- iterating over arrays -->
<div v-for="x of xs"></div>
<div v-for="(x, i) of xs"></div>

<!-- iterating over object keys -->
<div v-for="key in object"></div>
<div v-for="key of Object.keys(object)"></div>

<!-- iterating over objects -->
<div v-for="[key, value] of Object.entries(object)"></div>
<div v-for="([key, value], i) of Object.entries(object)"></div>
<div v-for="value of Object.values(object)"></div>
<div v-for="(value, i) of Object.values(object)"></div>

<!-- iterating through range -->
<div v-for="(_, i) of Array.from({ length: 10 })"></div>
```

# Motivation

In Vue 2.x, `v-for` does some "magic" to make iterating over object / 
array easy for developers. I believe the "magic" was added because 
`Object.entries` and `Object.values` did not exist or in an unstable 
stage when `v-for` was first implemented. 

By unifying the API for `v-for` and Javascript's `for` / `forEach`, we 
make developer's life easier because:

1. For complete beginners, they only need to care about one way of
   `for`-looping.
2. For Javascript developers who are new to Vue, they will feel at home
   when using `v-for`.

# Detailed design

Implement `v-for` so that it maps directly to `for`-loop / `forEach` in 
Javascript.

I refer to the following as "magic" because they do not behave the same
as Javascript's `for`. I believe they should be removed in Vue 3.x:

1. Iterating over objects with `for-in` (`v-for-in-object`)
```html
<div v-for="(value, key, index) in object"></div>
```

2. Iterating over array with `in` (`v-for-in-array`)
```html
<div v-for="x in xs"></div>
```

3. Iterating over number (`v-for-in-number`)
```html
<div v-for="i in 10"></div>
```

# Drawbacks

- Upgrading from Vue 2.x requires a lot of changes depending on how
  heavily `v-for-in-object` and `v-for-in-array` and `v-for-in-number`
  are used throughout the project.
- More verbose when using `v-for`.
- Less performant as it is required to create intermediate list to 
  iterating through entries in objects

# Alternatives

We can even go further and remove the `index` passed as the second
argument to make the `v-for` exactly the same as Javascript's `for`.

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
<div v-for="([key, value], index) of Object.entries(object)"></div>
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
<div v-for="i of Array.from({ length: 10 }, (_, i) => i + 1)"></div>
```
