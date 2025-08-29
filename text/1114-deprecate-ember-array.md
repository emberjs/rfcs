---
stage: ready-for-release
start-date: 2025-06-13T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - data
  - framework
  - learning
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1114'
project-link:
---

# Deprecating Ember Array

## Summary

Deprecate `Ember.Array`, `Ember.MutableArray`, `Ember.ArrayProxy`, `Ember.NativeArray`, `Ember.A`,
`Ember.Enumerable` and Array computed macros.

## Motivation

Now that Ember has a proper tracking system and the availability of `tracked-built-ins` we no longer
need our own custom arrays. In addition, much of the internal implementation relies upon Mixins which
we also hope to deprecate in the future.

## Transition Path

All uses of custom Arrays should be replaced with native Arrays and `tracked-built-ins`.
Additionally, computed macros should be updated to use tracked getters.

Given how estabished these patterns are, we probably want to provide an addon that brings
back at least some of the functionality to aid migration.

## Exploration

To validate this deprecation, I've tried removing EmberArray in this PR:
https://github.com/emberjs/ember.js/pull/20921

## How We Teach This

We should replace any references in the guides with native arrays or `TrackedArray`.

## Drawbacks

This functionality is old and very well established, removing it will not be without pain
to many users.

## Alternatives

Deprecate just `Ember.A`: #864

## Unresolved questions

None

## Notes

This overlaps with #1112 which also seeks to deprecate `Ember.ArrayProxy`.
