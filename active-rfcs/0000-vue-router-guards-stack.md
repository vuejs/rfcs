- Start Date: 2019-11-15
- Target Major Version: 3.x
- Reference Issues: -
- Implementation PR: -

# Summary

Allow a guard stack in global guards or in-route "beforeEnter" guard. Inspired in how `Laravel` handle this "problem".


# Basic example

The main ideas are:

1. Allow to define `beforeEach`, `afterEach` and `beforeResolve` in VueRouter setup. This is for not doing 1 by 1 and clearly look at the order of the guards stack.


```javascript
const router = new VueRouter({
  ...,

  // As array
  beforeEach: [
    (to, from, next) => { next() },
    (to, from, next) => { next() }
  ],

  // As Function
  beforeResolve: (to, from, next) => { next() },

  // As array, again
  afterEach: [
    (to, from) => { ... }
  ],

})

```

2. Allowing to define a stack in `beforeEnter` at route definition level:

```javascript
import needsToBeAuthenticated from '@/router/guards/needsToBeAuthenticated'
import needsToBeAuthorized from '@/router/guards/needsToBeAuthorized'

const routes = [
  {
    path: '/authenticated',
    component: () => import('@/pages/Authenticated'),
    beforeEnter: needsToBeAuthenticated,
  },
  {
    path: '/authorized',
    component: () => import('@/pages/Authorized'),
    beforeEnter: needsToBeAuthorized,
  },
  {
    path: '/authenticated-and-authorized',
    component: () => import('@/pages/AuthenticatedAndAuthorized'),
    beforeEnter: [
      needsToBeAuthenticated,
      needsToBeAuthorized
    ],
  }
]
```

# Motivation

The motivation behind this feature is reflected in the example above. In all my projects I have faced the same problem again and again. 

I have a guard that checks something. That guard can be reused in all other routes. But when I have a route that needs more checks, I find myself doing one of the following options:

1. Extract "unnecessarily" the guard behaviour to be able to call it from another guard.
2. Some weird router pattern matching in one `beforeEach / afterEach` guard to apply the desired behaviour.

When I use in (1) the word "unnecessarily" is because, in my opinion, I think the guard should be a simple function that checks something and then says yes or no. If I want to reuse this check I have to extract it to somewhere because of the implementation design of the router. I consider this a bit too overengineered for using a simple check.

I have found myself having a guard that simply calls a function from another module because the inner behaviour is extracted in another module for the sake of reuse that behaviour in another guard.

Allowing stacks, a guard can be defined and encapsulated in a function. It'll already extracted in its own function/module, and to use it you only need to push it to a stack.

# Detailed design

The design can be divided in two steps that can be released in different stages.

#### in-route `beforeEnter` guard

The `beforeEnter` option in the route now will accept a function or an array. This will allow backwards compatibility.

```javascript
[
  // As Array
  {
    ...,
    beforeEnter: [
      (to, from, next) => { next() },
      (to, from, next) => { next() }
    ],
  },

  // As Function. Current behaviour.
  {
    ...,
    beforeEnter: (to, from, next) => { next() },
  }
]

```

The definition of the guards will remain the same, including the priority based on order and the rules applied to `next` function.

#### VueRouter setup guards.

Allowing to define the stack inside the constructor options of the VueRouter looks, in my opinion, clearer. It'll allow too to reuse some guards in another life cycles. (Maybe a guard that is executed both `beforeEach` and `afterEach` for measurement purposes)

I understand this is totally subjective and can be dropped from this RFC if it's not a real issue.

# Drawbacks

Due this is not a breaking change, I consider that this feature doesn't trigger any real drawback. More or less, we've worked with backend frameworks where this behaviour is already implemented. (Laravel, Express, etc ...) 

# Alternatives

The alternative is to leave as it is, and extract/refactor when needed.

# Adoption strategy

This can be added to the documentation as an opt-in feature. That's all.

# Unresolved questions

Part 2 of this RFC, the VueRouter setup options, could be confusing if anyone uses both of the API's (setup options and `router.{hook}`). While the Vue Router can handle this situation, the user may not be clear which one is executed before another.

```javascript
import guard3 from 'somewhere'

const router = new VueRouter({
  beforeEach: [
    function guard1 (to, from, next) { next() },
    function guard2 (to, from, next) { next() }
  ]
})

router.beforeEach(guard3)
```

Although the order is clear, it may be confusing to anyone that may ask why use both APIs and not simply choose one. Of course, the second API, must remain, to be able to push guards from somewhere in our code.
