---
stage: accepted
start-date: 2025-05-13T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
  - learning
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

<!-- Replace "RFC title" with the title of your RFC -->

# Deprecate Ember Vendor Bundles

## Summary

The published ember-source package contains two different builds of Ember. This RFC deprecates the older one.

## Motivation

We don't want to to forever maintain the legacy, AMD-specific bundled copies of Ember that ember-cli traditionally appends to vendor.js. Instead, we want everybody consuming Ember as a modern ES module library.

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
> familiar with the framework to understand, and for somebody familiar with the
> implementation to implement. This should get into specifics and corner-cases,
> and include examples of how the feature is used. Any new terminology should be
> defined here.

> Please keep in mind any implications within the Ember ecosystem, such as:
>
> - Lint rules (ember-template-lint, eslint-plugin-ember) that should be added, modified or removed
> - Features that are replaced or made obsolete by this feature and should eventually be deprecated
> - Ember Inspector and debuggability
> - Server-side Rendering
> - Ember Engines
> - The Addon Ecosystem
> - IDE Support
> - Blueprints that should be added or modified

- Introduce an optional feature that opts into the new behavior (this is not hard to implement, it's mostly just disabling some existing compat code)
- Deprecate not enabling the optional feature

## How we teach this

> What names and terminology work best for these concepts and why? How is this
> idea best presented? As a continuation of existing Ember patterns, or as a
> wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
> re-organized or altered? Does it change how Ember is taught to new users
> at any level?

> How should this feature be introduced and taught to existing Ember
> users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

- Need to explain what addon authors should do if they were accessing Ember from code in treeForVendor.
- Explain that this whole thing is a no-op for Embroider users with `staticEmberSource:true` or `@embroider/core >= 4.0`.

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Ember,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

This RFC will cover ember.debug.js and ember.prod.js. It could also cover ember-template-compiler.js, or that can be a separate RFC.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
> TBD?
