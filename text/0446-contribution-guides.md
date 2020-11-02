---
Start Date: 2019-02-14
Relevant Team(s): Learning
RFC PR: https://github.com/emberjs/rfcs/pull/446
Tracking: https://github.com/emberjs/rfc-tracking/issues/39

---

# Contribution Guides

## Summary

This RFC proposes the creation of an official **Contribution Guide** that aims to improve the discoverability of Ember-related projects that require help by the community and outlines the general contribution workflow for these projects.


## Motivation

In the past year alone, [43](https://github.com/emberjs/ember.js/graphs/contributors?from=2018-01-01&to=2019-01-01&type=c), [34](https://github.com/ember-cli/ember-cli/graphs/contributors?from=2018-01-01&to=2019-01-01&type=c), [29](https://github.com/emberjs/data/graphs/contributors?from=2018-01-01&to=2019-01-01&type=c), [81](https://github.com/ember-learn/guides-source/graphs/contributors?from=2018-01-01&to=2019-01-01&type=c) and [31](https://github.com/emberjs/website/graphs/contributors?from=2018-01-01&to=2019-01-01&type=c) people made contributions to the [core framework library](https://github.com/emberjs/ember.js), the [Ember CLI](https://github.com/ember-cli/ember-cli), Ember’s [Data library](https://github.com/emberjs/data), the source for [the official framework Guides](https://github.com/ember-learn/guides-source), and the [project website](https://github.com/emberjs/website), respectively. An already impressive amount of developers decided to dedicate their time and effort to contribute to Ember - a project, that has been actively supported, maintained and developed by a strong open-source community for over seven years. Besides the code contributions to the main project repositories, the community can also look back on years of immense effort that has been put into the development of community-maintained packages which resulted in the vast Ember addon ecosystem as we know it today, as well as independent documentation and learning resources.

We are confident that there is an even greater potential for the community to contribute and that this can be unlocked by facilitating the contribution process - especially in regards to the on-boarding process for first-time contributors to Ember. Today, new community members oftentimes need to spend some time and effort to find a suitable project that allows them to apply a) their skills efficiently and b) is in their eyes worthwhile.

This RFC proposes to create and release a Contribution Guide - a new website as part of the `emberjs.com` domain. This new site should allow contributors to find Ember-related projects to work on easily and should provide more information on the workflow for their potentially (but not necessarily) first open-source contribution.

## Detailed design

The new Contribution Guides should be as **beginner-friendly** as possible to accommodate the needs of those who might make their very first code contribution to an open-source project.

 The list of topics covered should therefore include:

- **A summary of the motivation of open-source and its meaning for Ember as an OSS project.** This aims to provide more context to those new to Ember and open-source in general about the purpose of the project and how Ember values a collaborative approach in the development of the framework.

- **A How-To for code contributions with a real-world example.** This can include tips on how to find and claim a suitable issue, an introduction to Git and how to send a pull request, tips for the review process about finding a reviewer, communicating effectively and submitting changes and easy-to-review chunks of contribution work.

- **A How-To for filing a new issue on an Ember project**

- **An issue finder functionality inspired by the [What Can I Do for Mozilla landing page](https://whatcanidoformozilla.org/#!/progornoprog/teach)**. This should lead users to a) a list of relevant issues on the [Ember Help Wanted App](https://help-wanted.emberjs.com/), b) a related strike team channel on the Ember Discord chat, c) a link to the description of an on-going initiative on the [Status Board](https://emberjs.com/statusboard/).

Any subjects covered in the Guides should be presented as text instruction at least, but each topic can be enhanced with relevant code examples and multi-media content (e.g. slides of a relevant community talk, video content among others).

To help with the integration of the issue finder on the Contribution Guides with the Ember Help Wanted app, this proposes to extend the Ember Help Wanted App with a search filter for the main programming language used in the project. To make the Contribution Guides easy to discover from Ember projects, each project's specific contribution guidelines (e.g. [Ember CLI's `CONTRIBUTING.md`](https://github.com/ember-cli/ember-cli/blob/master/CONTRIBUTING.md)) should cross link back to the new Contribution Guides site for reference.

To improve discoverability of the Contribution Guides via search engines, it is also proposed to host the Guides on their own Ember sub domain: **contribute.emberjs.com**


## Alternatives

An alternative would be to not create a dedicated Contribution Guide and instead refer those that want to contribute to [the related section in the Ember Guides](https://guides.emberjs.com/release/contributing/adding-new-features/). It would also be possible to expand this section of the Ember Guides further with more content we'd like to share with contributors.

Also, the Ember Help Wanted App already works in favour of one of the goals of this proposal which is increasing the discoverability of issues that require contribution help. Instead of creating a new, dedicated website for the Contribution Guides, it would also make sense to progressively enhance the Help Wanted App with relevant content on contribution, e.g. with a short tutorial on the contribution workflow or How-Tos on other kinds of contribution work.

## Unresolved questions

Although this RFC already answers questions about the kind of content we’d like to see with an Contribution Guide for Ember, it is not clear yet, how the infrastructure for this guide looks like. Some of related questions are:

- Should the Guide be another Ember app, e.g another dedicated Guides app, in a similar to the official Ember and [Ember CLI Guides](https://github.com/ember-learn/cli-guides)? Making it an Ember app might in return help community members to contribute to it since they’re most likely already familiar with Ember. Choosing a similar format as the one of the Ember CLI Guides comes with the additional benefit of making it particularly easy to contribute to the Guide’s content since this only requires an understanding of Markdown, lowering the entry hurdle for a first-time contribution

- If we decide to not create another Guides app for the Contribution Guides, how should the design for the new website look like?

- This RFC only considered content related to code contributions relevant for the Contribution Guides. This excludes other types of important contribution work to Ember and the community, as event organisation, blogging, release management and many other topics. Should the Guides be expanded by relevant information about other types of contribution work as we iterate on it? This could include How-Tos for organising meetups, a collection of workshop materials for reuse, an introduction to the RFC process and other pieces of information
