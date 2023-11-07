---
stage: accepted
start-date: 2023-11-07T00:00:00.000Z
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
  accepted: https://github.com/emberjs/rfcs/pull/986
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

# Unify Ember and Glimmer git repos 

## Summary

We should combine glimmer-vm, glimmer.js, and ember-source repos into a single repo so we can iterate on them faster.
Monorepo tooling is in a good spot these days, and we can manage all projects at once, with their difference release schedules.


## Motivation

Iterating on Ember and Glimmer today is too hard. It shouldn't be. Today, we have to either link or do a series of publishes to verify that Glimmer(VM) works in ember-source. Linking becomes hard/impossible as we get strict about dependencies and not  allowing incorrect dependency graphs.

## Detailed design

### The current state

#### GlimmerVM

Repo: https://github.com/glimmerjs/glimmer-vm

The family of projects in this repo are all released under the same version. Since all packages here a private to ember consumers, we probably don't need to maintain this. Compatibility with package managers would improve if we also get out of the pre-v1 SemVer "weird zone" as well.


#### Glimmer.JS

Repo: https://github.com/glimmerjs/glimmer.js

The `main` branch of this repo is v2-beta, which isn't what ember is using. We'll need to fix this. Since this project defines `@glimmer/component` and `@glimmer/tracking` and these packages are consumer-facing, we'll need to be mindful of how we manage releases with these packages, and assure semver is properly communicated.

#### `ember-source`

Repo: github.com/emberjs/ember.js/



### Where we want to go

A single repo

### How we get there

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this


This RFC is for internal restructuring of our code, and nothing needs to be changed in the guides or learning materials. However, there will be some education needed for folks working within these repos -- in particular, around 
> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

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
