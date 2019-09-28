- Start Date: 2019-09-28
- Target Major Version: 3.x
- Reference Issues: (fill in existing related issues, if any)
- Implementation PR: (leave this empty)

# Summary

- Make `computed` only `get` to return an readonly immutable object.

# Basic example

```js
{
    data(){
        return {
            a: 1,
            b: 2
        }
    },
    computed: {
        mixedObject(){
            return {
                a,
                b,
                c: a + b
            }
        }
    },
    methods(){
        incrementA(){
            //this.mixedObject.a++; // do not allow this
            this.a++
        }
    }
}
```


# Motivation

Simplifying the api, allowing an readonly computed to be change will create a pitfall and not easy to spot bugs.

Allowing to add properties to a readonly computed it will only work until the computed is refreshed.

This is also motivated by the wording in the new `composition-api` rfc, since the read only computed are described as `immutable` ref object.

# Detailed design

Currently in 2.x you can do:
```js
export default {
  data() {
    return {
      test: 1,
    };
  },
  computed: {
    awesome() {
      return {
        test: this.test,
        test2: 1
      };
    }
  },

  methods: {
    increment() {
      // lets update all the underlining properties
      this.awesome.test++;
      this.awesome.test2++;
      this.awesome.test3 = 1;

      // console.log will output the `awesome` with `test3`
      console.log(this.awesome);
      /*
       * The render won't be updated, since the computed
       * is not reactive
       */
    },

    changeDataObject(){
      /*
       * this will reset the computed
       * to its original state, invalidating `test3` and 
       * any changes made to `awesome.test` are also ignored
       */
      this.test++;
    }
  }
}
```

# Drawbacks

The only drawback would do extra checks and enforce the computed to be readonly.

# Alternatives

Allowing the return object from a `readonly computed` to be `reactive`

# Adoption strategy

We can start showing warnings when trying to add/change a property in an readonly computed returned object.

Ideally will be better to create the proxy as immutable.

# Unresolved questions

- If we return an reactive object, should it be changed through the computed returned instance?
