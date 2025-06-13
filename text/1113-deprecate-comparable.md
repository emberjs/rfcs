---
stage: accepted
start-date: 2025-06-13T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1113
project-link:
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
-->

# Deprecating the `Comparable` Mixin

## Summary

Deprecate the `Comparable` Mixin.

## Motivation

For a while now, Ember has not recommended the use of Mixins. In order to fully
deprecate Mixins, we need to deprecate all existing Mixins of which `Comparable` is one.

## Transition Path

None. This will be removed without replacement.

## Exploration

To validate this deprecation, I've tried removing the `Comparable` Mixin from Ember.js in this PR:
https://github.com/emberjs/ember.js/pull/20924

## How We Teach This

We should remove all references from the guides.

## Drawbacks

Some users probably rely on this functionality. However, it's almost certainly
something that we don't need to keep in Ember itself.

## Alternatives

* Convert `Comparable` to a class decorator-style mixin.

## Unresolved questions

None