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

The family of projects in this repo are all released under the same version. Since all packages here are private to ember consumers, we probably don't need to maintain this. Compatibility with package managers would improve if we also get out of the pre-v1 SemVer "weird zone" as well.


#### Glimmer.JS

Repo: https://github.com/glimmerjs/glimmer.js

The `main` branch of this repo is v2-beta, which isn't what ember is using. We'll need to fix this. Since this project defines `@glimmer/component` and `@glimmer/tracking` and these packages are consumer-facing, we'll need to be mindful of how we manage releases with these packages, and assure SemVer is properly communicated.

#### `ember-source`

Repo: https://github.com/emberjs/ember.js


### Where we want to go

A single repo.
```
github.com/emberjs/ember.js/
    .github/
    package.json
    packages/
        @glimmer/
            ...
        @ember/
            ...
        ember-source/
            ...
    tests/
        high level/
        integration tests/
    dev/
        packages/
            configs/
            test-infra/
            etc
            
```

### How we get there

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this


This RFC is for internal restructuring of our code, and nothing needs to be changed in the guides or learning materials. However, there will be some education needed for folks working within these repos -- in particular, around monorepo management, dependent builds, etc.

## Drawbacks

Monorepos do require some in-repo infra, but all our repos already have a bunch of custom infra already.
The goal would be to use the least amount of custom stuff as possible.

## Alternatives


## Unresolved questions

