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
  accepted: https://github.com/emberjs/rfcs/pull/1112
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

# Deprecating Ember Proxies

## Summary

Now that Native Proxy is available in all supported environments, we can deprecate
Ember's custom Proxy handling provided by `ArrayProxy`, `ObjectProxy`, and `PromiseProxyMixin`.

## Motivation

These patterns were introduce to Ember prior to the availability of Native Proxy. Since
Native Proxy is now available in all supported environments, we can deprecate these
patterns in favor of the native Proxy API.

Additionally, we would like to deprecate Mixins in the future necessitating that we first
deprecate `PromiseProxyMixin`.

## Transition Path

There is no direct migration path for these. Code that relies upon this behavior will have to be rewritten.

See the deprecation guide (PR'd here: https://github.com/ember-learn/deprecation-app/pull/1405 )

## Exploration

To validate this deprecation, I've tried removing the assocaited functionality in this PR:
https://github.com/emberjs/ember.js/pull/20918

## How We Teach This

We should remove references to these patterns from the guides.

## Drawbacks

By not providing a direct replacement some users may have difficulty migrating.

## Alternatives

None

## Unresolved questions

None
