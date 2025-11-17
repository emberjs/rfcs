---
stage: accepted
start-date: # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
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

# Public API for render-tree-based contextual exploration 

## Summary

This RFC proposes a public API that enables safe experimentation with render-tree-based contextual data access patterns. The API provides scope-based synchronous access for all invokables (components, helpers, modifiers) across apps, engines, and addons.

## Motivation

The Ember community has long desired a Context API to share state across the render tree without prop drilling, but developers currently must resort to [abusing private Glimmer APIs](https://github.com/customerio/ember-provide-consume-context/blob/def6d34f639d56ebec1c7c8c888f86ec524b8688/ember-provide-consume-context/src/-private/override-glimmer-runtime-classes.ts#L1), creating upgrade risks and maintenance burdens. This RFC provides a public API that enables safe experimentation with Context patterns for use cases like theme systems, authentication state, and feature flags. The expected outcome is to establish a stable foundation for community exploration of contextual data access patterns while gathering real-world feedback to inform future official Context API design.

Additionally, this also enables exploration of providing _owner_ access to plain function helpers, modifiers, (all invokables).

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here. 

> Please keep in mind any implications within the Ember ecosystem, such as:
> - Lint rules (ember-template-lint, eslint-plugin-ember) that should be added, modified or removed
> - Features that are replaced or made obsolete by this feature and should eventually be deprecated
> - Ember Inspector and debuggability
> - Server-side Rendering
> - Ember Engines
> - The Addon Ecosystem
> - IDE Support
> - Blueprints that should be added or modified

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

- [ember-provide-consume-context](https://github.com/customerio/ember-provide-consume-context)
- [passing a third argument to component constructors that is the VM Stack](https://github.com/rtablada/ember-context-experiment/blob/main/app/components/UserName.gjs)

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
