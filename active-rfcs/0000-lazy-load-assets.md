- Start Date: 2020-07-06
- Target Major Version: 3.x
- Reference Issues: https://github.com/vuejs/vue-next/pull/1483
- Implementation PR: (leave this empty)

# Summary

When you create a Vue component, you pass an object with subcomponents that can be called inside.
This means that all subcomponents must be defined at creation time of the first component.
But if we passed a function instead of a object, components could be loaded only when and if
necessary. This would add lazy loading of components into Vue.

# Basic example

```javascript
var vm = Vue.createApp({
	setup: function() {
		return {};
	},
	getComponent: function(spellings, instance) {
		console.log('App GetComponent', spellings, instance);
		// example of spellings object:
		// {
		// 	"raw": "shift-foo",
		// 	"hyphenated": "shift-foo",
		// 	"camelized": "shiftFoo",
		// 	"PascalCase": "ShiftFoo"
		// }
		// In reality, you should return a component corresponding to the name in the object above
		return {
			template: '<div><p>She\'s like the {{state.shesLikeThe}}</p></div>',
			setup: function() {
				var state = Vue.reactive({
					shesLikeThe: 'wind'
				});
				return {
					state: state
				};
			}
		}
	}
});
```

# Motivation

In order to have an architecture of an app with hundreds of organized components which are loaded on demand, and this is the crucial part: **without a build step.** I understand that in a high traffic application, you need the optimization opportunities that a build step provides.

However, in my company case, and especially because of the back-end engineers, they want to be able to give maintenance and support to our shipped applications. We're short staffed with front-end engineers, só they leave only the heavy development for us. We have some applications that require constant changes because of business requirements, and they want to do them quickly.

They are used to the old way of doing things, where you had one page with some jquery commands, and they think they could deliver the necessary changes much faster back then, without a build step.

In our case, the performance of our applications is very satisfatory to our clients, but we need to deliver the changes faster, so our bottleneck is in our development team's time. But at the same time, we recognize that to develop a modern and complex application, but in an organized fashion, we need to break It down into components, defining a very clear separation of concerns.

The only way we can develop these components separately and deploy them without a build step is by loading them asynchronally on runtime. We've defined a naming pattern and a structure, a way in which our components must be created. With these rules, we can tell Vue how to load our components without having a pre-defined component name list, which, without a build step, would have to be manually mantained.

There is a reason I believe this is a simple change, and one that does not break backward compatibility - I'm simply defining one function which gives the app an opportunity to resolve a component constructor in runtime. It is really a shame that what I need to do really isn't possible to achieve without modificating the framework itself.

The API for constructors is pretty well established, it is the same for named components. So I think this really doesn't add any complexity to future updates in the framework itself, it is not a costly feature, It is designed with the absolute bare minimum requirements so I can build upon It.

# Detailed design

Here's the implementation: https://github.com/vuejs/vue-next/pull/1483

And here's a real world example: https://github.com/arijs/vue-next-example

The changes are minimal. All we need are automated tests, but the implementation (well, the Vue 2.x version) has already been used by me in many sites I developed.

I could accept an API to expose this capability to plugins and not directly to component options. However, this would be more complex, and I couldn't figure out how to do that easily.

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
  - The changes are few. However, one issue that could be more discussed is the `spellings` object - I don't simply pass the raw string name of the component because Vue itself does this normalization, converting names to camelCase, PascalCase and Hyphenated. Maybe instead of always generating all the spellings, these convert functions could be exposed.
- whether the proposed feature can be implemented in user space
  - Unfortunately no.
- the impact on teaching people Vue
  - This would not substitute the current way of registering components in a object map, so nothing would be lost.
- integration of this feature with other existing and planned features
  - It is already integrated and tested in my fork with the applications I've built. Would be happy if this was already planned.
- cost of migrating existing Vue applications (is it a breaking change?)
  - Not a breaking change, so no need to migrate.

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

I guess the alternative is running the behemoth of webpack, which is enormously complex and obfuscates my code even during development. It is not a simple dislike of webpack, I literally can't put a breakpoint on my components' code when it is built by webpack.

# Adoption strategy

If we implement this proposal, how will existing Vue developers adopt it? Is
this a breaking change? Can we write a codemod? Can we provide a runtime adapter library for the original API it replaces? How will this affect other projects in the Vue ecosystem?

They adopt it by building a new application without webpack. Most people now use it, but many simply use it because it's the default and don't concern themselves with simplicity.

# Unresolved questions

