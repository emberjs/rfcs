---
stage: ready-for-release
start-date: 2025-06-20T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1117'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1141'
project-link:
---

# Deprecate Classic Classes

## Summary

Deprecate the Classic Class system.

## Motivation

For quite a while now, Ember has recommended the use of Native Classes over the Classic Class
system. The one remaining sticking point has been Mixins. If we
[Deprecate Mixins](https://github.com/emberjs/rfcs/pull/1116), then we can deprecate the Classic Class system.

## Transition Path

All uses of Classic Classes should be converted to Native Classes. Anything that cannot be converted will be deprecated first, such as [Mixins](https://github.com/emberjs/rfcs/pull/1116) and the [`observer` helper function](https://github.com/emberjs/rfcs/pull/1115).

The `@classic` decorator (and any other APIs or patterns used to enable classic class interop) will also be deprecated as part of this transition. All code should migrate to native class syntax and patterns.

## Exploration

To validate this deprecation, I've tried removing all of the Classic Class system in this PR:
https://github.com/emberjs/ember.js/pull/20923

## How We Teach This

We should remove all references from the guides.

## Drawbacks

Other than it being a lot of work, there are no drawbacks.

## Alternatives

None

## Unresolved questions

Should we keep `EmberObject` around as a home for some utility methods and sugar or remove it entirely?