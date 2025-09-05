---
stage: ready-for-release
start-date: 2025-06-20T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - framework
  - learning
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1116'
project-link:
---

# Deprecating Mixin Support

## Summary

Deprecate all Mixin support.

## Motivation

For a while now, Ember has not recommended the use of Mixins. We should actually fully
deprecate this.

## Transition Path

All existing public Ember mixins will be deprecated:

- [Ember.Mixin](https://github.com/emberjs/rfcs/pull/1111) (from `@ember/object/mixin`)
- [Ember.PromiseProxyMixin](https://github.com/emberjs/rfcs/pull/1112) (from `@ember/object/promise-proxy-mixin`)
- [Ember.Comparable](https://github.com/emberjs/rfcs/pull/1113) (from `@ember/object/comparable`)
- [Ember.Enumerable](https://github.com/emberjs/rfcs/pull/1114) (from `@ember/object/enumerable`)
- [Ember.Observable](https://github.com/emberjs/rfcs/pull/1115) (from `@ember/object/observable`)

For users that still want mixin-like functionality, we should recommend class
decorators. If this feels less ergonimic that we would desire, we can consider
providing an addon that adds some sugar around this.

## Exploration

To validate this deprecation, I've tried removing all Mixins in this PR: https://github.com/emberjs/ember.js/pull/20923

## How We Teach This

We should remove all references from the guides and provide a [deprecation guide](https://github.com/ember-learn/deprecation-app/pull/1408).

## Drawbacks

Some users probably rely on this functionality. However, it's almost certainly
something that we don't need to keep in Ember itself.

## Alternatives

None

## Unresolved questions

None