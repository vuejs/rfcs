- Start Date: 2020-01-29
- Target Major Version: 3.x
- Reference Issues: N/A
- Implementation PR: (leave this empty)

# Summary

While there are many similarities, as it stands the API docs for the [various](https://vuejs.org/v2/api/) [official](https://vuex.vuejs.org/api/) [Vue-related](https://router.vuejs.org/api/) [repositories](https://cli.vuejs.org/config/) are still somewhat inconsistent.

Examples of this include:

1. [JSDoc-style signatures](https://vuejs.org/v2/api/#Vue-extend) (`{Object} options`) being used in one place while more [TypeScript-looking signatures](https://vuejs.org/v2/api/#optionMergeStrategies) (`{ [key: string]: Function }`) can be found in others
2. Then we have vue-router which sometimes provides [TypeScript signatures](https://router.vuejs.org/api/#scrollbehavior) and at others just does its own thing [with list items](https://router.vuejs.org/api/#v-slot-api-3-1-0), or within [fenced code blocks](https://router.vuejs.org/api/#to)
3. Vue likes to put the [usage examples within a list item](https://vuejs.org/v2/api/#silent) whereas [Vuex just puts them inline](https://vuex.vuejs.org/api/#subscribeaction), with the rest of the text
4. Vuex has the "new in version / since version" [in a blockquote](https://vuex.vuejs.org/api/#subscribeaction) underneath the signature, Vue puts them [in a blockquote](https://vuejs.org/v2/api/#inheritAttrs) except it's always at the very top, meanwhile vue-router likes to [put it in parentheses in the heading](https://router.vuejs.org/api/#v-slot-api-3-1-0)
5. [Vue](https://vuejs.org/v2/api/#Built-In-Components) has a [dedicated section for built-in components](https://vuejs.org/v2/api/#Built-In-Components) whereas [vue-router has them as top-level headings](https://router.vuejs.org/api/#router-link)

# Motivation

## Why are we doing this?

The above may just seem like nitpicking about minor details, but in the context of trying to parse these documents into a structured format such as JSON it's actually quite significant. Our own use case, then, is to allow API docs lookups via the Discord bot but due to the above points, and others, it proved problematic.

I parsed the API docs markdown files into Abstract Syntax Trees and then tried to work with that but it proved impossible, or at least extremely difficult.

For now I have instead resorted to a _semi-automated_ approach (with a lot of manual touching up) but it's a relatively time-consuming process.

The thing is, if you look closely, you'll see that things _already are_ vaguely standardised! Most of the repositories follow the convention of having a list item for each of: `type`, `default` and `required`, for instance.

## Which use cases does it support?

1. It will provide the documentation with a greater level/feeling of _cohesion_, which makes sense since vuex, vue-router and so on _are_ all part of a greater whole
2. Anyone working on multiple Vue-related API docs won't ever need to perform a context switch, when going from one project's docs to the next, since there will be only one way of writing API docs
3. If someone, like me, wants to parse the API markdown files into a more structured format, to build external/unofficial/third-party docs-related tools/tooling, e.g. an unofficial docs website, an API explorer mobile app, a Discord/Slack bot, then they can do so

# Detailed design

TBD

# Drawbacks

The main drawback is of course that there are _probably not_ going to be (m)any others who want to do what I'm doing. In that case, it's in a way just creating more work for the docs teams to save me some work, which make zero sense.

I am content with sticking with my somewhat manual approach - if I were to keep a close eye on API changes (by monitoring releases) then it wouldn't be _too_ much work to stay aligned.

That said, I'm also willing to help tackle this and to do some of the grunt/chore work that would be involved.

# Alternatives

## Fully generated API docs

I had also considered the prospect of the API docs being generated from JSDoc blocks and/or TS typings, but I have heard whispers that some people on the docs team don't want generated docs, despite that it seems to work fairly well for other projects such as [Quasar](https://quasar.dev/vue-components/ajax-bar#QAjaxBar-API) ([raw markdown](https://raw.githubusercontent.com/quasarframework/quasar/dev/docs/src/pages/vue-components/ajax-bar.md))

As you can see, their guide is markdown-based, like yours, but the _API docs specifically_ are a Vue component, embedded in the guide markdown, which [loads some JSON](https://github.com/quasarframework/quasar/blob/dev/docs/src/components/DocApi.vue#L305). I am not sure if their JSON is generated from JSDoc blocks/typings or handcrafted, however.

Consider also that since all of the official docs are using VuePress, there would not need to be duplicate efforts on docs, with an approach like this, if some kind of consensus could be reached which covers the various project's documentation-related needs.

This would of course be a **lot more work** than simply trying to standardise the existing Markdown-based API docs.

## Full-text searching of Markdown files

Someone mentioned that for our use-case we could just use something like [Solr](https://lucene.apache.org/solr/) (or [Lunr](https://lunrjs.com/)), without the need to parse/generate any data, and indeed we are using Lunr for searching through the **guides**, but API docs are very different to textual documents, and from my experiments I don't think Lunr nor Fuse.js (nor any other solution) is going to do a particularly good job _for API docs specifically_.

# Adoption strategy

N/A

# Unresolved questions

I haven't yet worked on a "detailed design". Of course we would need to find a standard which can support the needs of all the various projects, after all a router, a state management solution, a CLI (etc.) are very different beasts and that will be reflected in their APIs/API docs.

I'm happy to attempt that, especially if the docs folks don't have time but I think that that process would need some input or oversight from them.
