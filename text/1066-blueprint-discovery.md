---
stage: accepted
start-date: 2024-12-24T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
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

<-- Replace "RFC title" with the title of your RFC -->
# v2 blueprint discovery protocol

## Summary

Since the beginning of ember-cli we have had blueprints available for developers to create new apps, new addons, and a number of different entities for your existing apps such as Components, Routes, Tests, etc. A lesser known feature of the blueprint system is how they are discovered, i.e. when you run `ember g component` how does ember-cli know how to find the addon that is providing that `component` blueprint. 

This RFC intends to formalise the protocol that ember-cli uses to discover blueprints with a new protocol that will be a lot faster than the previous discovery process and more future proof. This RFC also proposes that we deprecate the old blueprint discovery process. 

## Motivation

There are a number of motiviations for this RFC that i will go into in more detail in this section, but for now the main motivators are

- v2 addons have no way to provide blueprints TODO actually check this 
- blueprint generation can feel very slow
- there is no clear way to disambiuate blueprints that are being provided by multiple addons

### Addon loading

To understand some of the challenges around blueprints loading we have to first understand some of the protocol that ember-cli uses to load addons (specifically non v2 addons). Whenever you run an ember-cli command that needs to have information about addons (such as `ember generate` or `ember serve`) ember-cli needs to go through the addon discovery process. This process can be very complicated but a simplified flow might be as follows:

TODO simplified flow of addon discovery 

 

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

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
