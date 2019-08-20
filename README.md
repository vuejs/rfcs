# Vue RFCs

## What is an RFC?

The "RFC" (request for comments) process is intended to provide a
consistent and controlled path for new features to enter the framework.

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put
through a bit of a design process and produce a consensus among the Vue
core team and the community.

## The RFC life-cycle

An RFC goes through the following stages:

- **Pending:** when the RFC is submitted as a PR.
- **Active:** when an RFC PR is merged and undergoing implementation.
- **Landed:** when an RFC's proposed changes are shipped in an actual release.
- **Rejected:** when an RFC PR is closed without being merged.

[Pending RFC List](https://github.com/vuejs/rfcs/pulls)

## When to follow this process

You need to follow this process if you intend to make "substantial"
changes to one of the projects listed below:

- [Vue core](https://github.com/vuejs/vue)
- [Vue Router](https://github.com/vuejs/vue-router)
- [Vuex](https://github.com/vuejs/vuex)
- [Vue CLI](https://github.com/vuejs/vue-cli)

We are limiting the RFC process for these repos to test out the process in a more manageable fashion, and may expand it to cover more projects under the `vuejs` organization in the future. For now, if you wish to suggest changes to those other projects, please use their respective issue lists.

What constitutes a "substantial" change is evolving based on community norms, but may include the following:

- A new feature that creates new API surface area
- Changing the semantics or behavior of an existing API
- The removal of features that are already shipped as part of the release channel.
- The introduction of new idiomatic usage or conventions, even if they do not include code changes to Vue itself.

Some changes do not require an RFC:

- Additions that strictly improve objective, numerical quality criteria (speedup, better browser support)
- Fixing objectively incorrect behavior
- Rephrasing, reorganizing or refactoring
- Addition or removal of warnings
- Additions only likely to be _noticed by_ other implementors-of-Vue, invisible to users-of-Vue.

If you submit a pull request to implement a new feature without going
through the RFC process, it may be closed with a polite request to
submit an RFC first.

## Why do you need to do this

It is great that you are considering suggesting new features or changes to Vue - we appreciate your willingness to contribute! However, as Vue becomes more widely used, we need to take stability more seriously, and thus have to carefully consider the impact of every change we make that may affect end users. On the other hand, we also feel that Vue has reached a stage where we want to start consciously preventing further complexity from new API surfaces.

These constraints and tradeoffs may not be immediately obvious to users who are proposing a change just to solve a specific problem they just ran into. The RFC process serves as a way to guide you through our thought process when making changes to Vue, so that we can be on the same page when discussing why or why not these changes should be made.

## Gathering feedback before submitting

It's often helpful to get feedback on your concept before diving into the
level of API design detail required for an RFC. **You may open an
issue on this repo to start a high-level discussion**, with the goal of
eventually formulating an RFC pull request with the specific implementation
design.

## What the process is

In short, to get a major feature added to Vue, one must first get the
RFC merged into the RFC repo as a markdown file. At that point the RFC
is 'active' and may be implemented with the goal of eventual inclusion
into Vue.

* Fork the RFC repo http://github.com/vuejs/rfcs

* Copy `0000-template.md` to `active-rfcs/0000-my-feature.md` (where
'my-feature' is descriptive. don't assign an RFC number yet).

* Fill in the RFC. Put care into the details: **RFCs that do not
present convincing motivation, demonstrate understanding of the
impact of the design, or are disingenuous about the drawbacks or
alternatives tend to be poorly-received**.

* Submit a pull request. As a pull request the RFC will receive design
feedback from the larger community, and the author should be prepared
to revise it in response.

* Build consensus and integrate feedback. RFCs that have broad support
are much more likely to make progress than those that don't receive any
comments.

* Eventually, the [core team] will decide whether the RFC is a candidate
for inclusion in Vue.

* An RFC can be modified based upon feedback from the [core team] and community. Significant modifications may trigger a new final comment period.

* An RFC may be rejected after public discussion has settled
and comments have been made summarizing the rationale for rejection. A member of the [core team] should then close the RFC's associated pull request.

* An RFC may be accepted at the close of its final comment period. A [core team] member will merge the RFC's associated pull request, at which point the RFC will become 'active'.

## Details on Active RFCs

Once an RFC becomes active then authors may implement it and submit the
feature as a pull request to the Vue repo. Becoming 'active' is not a rubber
stamp, and in particular still does not mean the feature will ultimately
be merged; it does mean that the core team has agreed to it in principle
and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is
'active' implies nothing about what priority is assigned to its
implementation, nor whether anybody is currently working on it.

Modifications to active RFC's can be done in followup PR's. We strive
to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect
every merged RFC to actually reflect what the end result will be at
the time of the next major release; therefore we try to keep each RFC
document somewhat in sync with the language feature as planned,
tracking such changes via followup pull requests to the document.

## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the
RFC author (like any other developer) is welcome to post an
implementation for review after the RFC has been accepted.

An active RFC should have the link to the implementation PR listed if there is one. Feedback to the actual implementation should be conducted in the implementation PR instead of the original RFC PR.

If you are interested in working on the implementation for an 'active'
RFC, but cannot determine if someone else is already working on it,
feel free to ask (e.g. by leaving a comment on the associated issue).

## Reviewing RFC's

Members of the [core team] will attempt to review some set of open RFC
pull requests on a regular basis. If a core team member believes an RFC PR is ready to be accepted into active status, they can approve the PR using GitHub's review feature to signal their approval of the RFC.

**Vue's RFC process owes its inspiration to the [React RFC process], [Rust RFC process] and [Ember RFC process]**

[React RFC process]: https://github.com/reactjs/rfcs
[Rust RFC process]: https://github.com/rust-lang/rfcs
[Ember RFC process]: https://github.com/emberjs/rfcs
[core team]: https://vuejs.org/v2/guide/team.html
