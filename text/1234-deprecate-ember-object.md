---
stage: accepted
start-date:
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
  accepted: https://github.com/emberjs/rfcs/pull/1234 
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

# Deprecate EmberObject 

## Summary

Deprecates `EmberObject` in a way that is initually opt-in, so people can more gradually prepare their codebase for the removal of `EmberObject`. 


## Motivation

`EmberObject` has had heavy use in the early days of Ember (pre-JavaScript having classes), and since classes shipped in 2015, the need for `EmberObject` has greatly diminished. 

Removing `EmberObject` is one of the last steps in coercing codebases to be plain modern JavaScript.

## Transition Path

Unlike previous deprecations, this is targeting Ember 9, and will have a feature flag that removes all behavior related to `EmberObject`. This does require a lot of internal implementation in `ember-source`, but is needed anyway for the removal of `EmberObject`, ultimately.

This will be the first deprecation that users will be able to preview the removal off.

Leading up to v8, the feature flag will be "off":
- deprecation logged for not having this feature flag "on" 
- EmberObject and all related APIs are still usable

With the release of v8, and leading up to v9, the feature flag will be "on" by default:
- EmberObject and related APIs are not usable, due to the feature flag removing all of the implementation
- if users wish, the feature flag can be flipped back off, which brings back the EmberObject behavior along with the deprecation

At `ember-source` v9, `EmberObject` is removed fully along with the feature flag.


> [!NOTE]
> This includes `@computed`, as `@computed` is part of the "Ember Object Model" of reactivity.


Internally, implementation would likely be similar to how Mixins were initially deprecated -- copied to an "internal" file, and then the "public" version of `EmberObject` would override `init`, and provide the deprecations. 

On the internal copy of `EmberObject`, we deprecate all the methods (`get`, / `set` / etc), so that the deprecations flow through to other framework classes such as `Route`, `Controller`, `Service`, etc.

## How We Teach This

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?
Does it mean we need to put effort into highlighting the replacement
functionality more? What should we do about documentation, in the guides
related to this feature?
How should this deprecation be introduced and explained to existing Ember
users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

## Drawbacks

keeping EmberObject is a drawback, because of the dozens of KB that come along with it.

all codebases with old code probably have some usage of EmberObject remaining, so they need to migrate.

## Alternatives

- do nothing

## Unresolved questions

n/a
