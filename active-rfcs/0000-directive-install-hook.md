- Start Date: 2021-08-02
- Target Major Version: 3.x
- Reference Issues: 
- Implementation PR:  https://github.com/vuejs/vue-next/pull/4235

# Summary
add  `install` hook for a custom directive

# Basic example

```js
export default {
   directives: {
      MyDir: {
         install(){ return { mounted(){}, unmounted(){} } 
      } 
   } 
}
```

# Motivation

In current state when a function is passed as a directive, vue will register the function 
to  `mounted` and `updated` hooks only.
additionally there is no way to tell which hook invoked the function, or register to other hooks.

having an `install` hook will provide 2 main advantages. 
1. allows for function based directive, that can target any of the supported directive hooks 
2. having access to the directive early enough, that a function can be used to create the directive dynamically, 
with access to the directive binding(instance, value, etc) 




Why are we doing this? What use cases does it support? What is the expected
outcome?

1. The current function based directive is limited to 2 fixed hooks, and no ability to know the life cycle phase
when the function is invoked.

2. consider provide/inject api, a parent component can actually provide a directive as an `install` function 
where the child can then *inject and use. 
(* currently inject() is not callable in a directive hook, however as a work around, a child can "resolve" the provided value via. `instance.$.provides` ) 
which make this use case doable. even without full support from the inject functionality.



# Detailed design


https://github.com/vuejs/vue-next/pull/4235


```js
export default {
   directives: {
      myDir: {
          install(binding : DirectiveBinding): ObjectDirective {
            return { mounted(){}, unmounted(){}, ... } 
          }
      }
   } 
}
```

if `install` hook is provided it will be called and the result will be merged into the directive hooks object.


# Drawbacks

 a function based directive might have additional overhead, however vue already supports a very limited function based directive.


# Alternatives
n/a


# Adoption strategy

The feature is additive and should not have any impact on existing code.


