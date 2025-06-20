---
stage: accepted
start-date: 2025-06-20T00:00:00.000Z
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
  accepted: https://github.com/emberjs/rfcs/pull/1116
project-link:
---

# Deprecating Mixin Support

## Summary

Deprecate all Mixin support.

## Motivation

For a while now, Ember has not recommended the use of Mixins. We should actually fully
deprecate this.

## Transition Path

All existing public Ember mixins will be deprected:

- [Ember.Mixin](https://github.com/wagenet/rfcs/pulls/1111)
- [Ember.PromiseProxyMixin](https://github.com/wagenet/rfcs/pulls/1112)
- [Ember.Comparable](https://github.com/wagenet/rfcs/pulls/1113)
- [Ember.Enumerable](https://github.com/wagenet/rfcs/pulls/1114)
- [Ember.Observable](https://github.com/wagenet/rfcs/pulls/1115)

For users that still want mixin-like functionality, we should recommend class
decorators. If this feels less ergonimic that we would desire, we can consider
providing an addon that adds some sugar around this.

## Exploration

To validate this deprecation, I've tried removing all Mixins in this PR: https://github.com/emberjs/ember.js/pull/20923

## How We Teach This

We should remove all references from the guides.

## Drawbacks

Some users probably rely on this functionality. However, it's almost certainly
something that we don't need to keep in Ember itself.

## Alternatives

None

## Unresolved questions

None