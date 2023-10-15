- Start Date: 2019-02-01
- Target Major Version: 2.x
- Reference Issues: [vuejs/vue#7943](https://github.com/vuejs/vue/issues/7943)
- Implementation PR: (leave this empty)

# Summary

Allow aliasing props, to have different external and internal names (some languages call these "argument labels"):

```js
props: {
    externalName: {
        as: 'internalName'
    }
}
```

The `as` nomenclature is of course open for bikeshedding.

# Basic example

In this example (which is a modified version of the one [in Vue's docs](https://vuejs.org/v2/guide/components-props.html)), the component has a `counter` prop as part of its external API. It's aliased as `initialCounter` within the component itself, to prevent clashing with its own `counter` data property:

```js
export default {
    props: {
        counter: {
            as: 'initialCounter'
        }
    },
    data () {
        return {
            counter: this.initialCounter
        }
    }
}
```

Here's how the component would be consumed:

```html
<the-component :counter="5" />
```

# Motivation

Since props are not to be mutated, it is recommended to define a local data property that uses the prop as its initial value.

1. Here is one example from Vue's docs:

    ```js
    props: ['initialCounter'],
    data() {
        return {
            counter: this.initialCounter
        }
    }
    ```

    This makes sense within the component itself, but it feels wrong for it to affect the external API:

    ```html
    <the-component :initialCounter="5" />
    ```

    Aliasing the prop locally (as outlined above) would allow the consumer to use the simpler `counter` name for the prop:

    ```html
    <the-component :counter="5" />
    ```

1. Here's another example from the docs:

    ```js
    props: ['size'],
    computed: {
        normalizedSize() {
            return this.size.trim().toLowerCase()
        }
    }
    ```

    This works, but now you have to always refer to the sanitized size as `normalizedSize` within your component.

    Using an alias, we can change it to:

    ```js
    props: {
        size: {
            as: 'rawSize'
        }
    },
    computed: {
        size() {
            return this.rawSize.trim().toLowerCase()
        }
    }
    ```

    which would allow us to refer to the normalized size as just `size` within our component, without affecting the component's external API.

## Prior art

External argument labels is something that is supported in many different languages.

Here's how they work [in Swift](https://docs.swift.org/swift-book/LanguageGuide/Functions.html):

```swift
func greet(person: String, from hometown: String) -> String {
    return "Hello \(person)! Glad you could visit from \(hometown)."
}

greet(person: "Bill", from: "Cupertino")
```

The second argument is externally named `from` and internally named `hometown`. This makes for very intuitive designs, as evidenced by many of Apple's library APIs using this convention.

In fact, JS itself also supports external argument labels (insofar as object destructuring is JS's support for named arguments). Here's the JS version of that Swift function:

```js
function greet({ person, from: hometown }) {
    return `Hello ${person})! Glad you could visit from ${hometown}.`
}

greet({ person: 'Bill', from: 'Cupertino' })
```

# Detailed design

**Introducing a new `as` key to prop options.**

If a prop's options specifies an `as` key, its value will be used as the internal name for the prop:

```js
props: {
    externalName: {
        as: 'internalName'
    }
}
```

If the `as` key is not specified, the internal name will be the same as the external name, as it has always been.

# Drawbacks

- The new syntax may be confusing to beginners. Seeing a property used in the component without seeing that property in either the original data object or the top-level of the `props` may be a little disconcering to newcomers.

# Alternatives

The alternative is to keep things as is, and always use the same prop name both internally and externally.

# Adoption strategy

Since this change is purely additive, there's nothing we would have to do to help with its adoption. It can be added to the docs, and people can use them wherever it makes sense in their projects.

# Unresolved questions

1. **Should the top-level key in the `props` object be the internal name or the external name?**

    This proposal uses the top-level key as the external name, with `as` denoting its internal name:

    ```js
    props: {
        externalName: {
            as: 'internalName'
        }
    }
    ```

    This makes it easy for consumers of the component to see at a glance which props they can pass in, without having to parse the prop's options.

    Another option would be to have the top-level key be the internal name:

    ```js
    props: {
        internalName: {
            as: 'externalName'
        }
    }
    ```

    This would make it easier when working _within_ the component to quickly see what props are available to be accessed.

    The proposal currently puts more emphasis on the consumer's glanceability, since you usually have more knowledge of a component while you're working on it.

1. **What should be the name of the key in the prop's options used for aliasing the prop?**

    This proposal uses `as`, but there are many different keys that could be considered (depending on whether it denotes an internal name or an external name).

    Here are some alternate names:

    - `alias`
    - `label`
    - `expose`
    - `internal`
    - `external`
