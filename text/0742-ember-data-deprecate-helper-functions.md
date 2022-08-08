---
start-date: 2021-04-23T00:00:00.000Z
release-date:
release-versions: 
teams: 
  - data
prs:
  accepted: https://github.com/emberjs/rfcs/pull/742
project-link: 
stage: accepted
---

# EmberData | Deprecate Helper Functions

## Summary

Deprecates the exported util functions `errorsHashToArray` `errorsArrayToHash`
and `normalizeModelName` that were recommended for deprecation by the [RFC#395 packages](https://github.com/emberjs/rfcs/blob/master/text/0395-ember-data-packages.md)

## Motivation

These utils are a legacy remnant of when parts of the codebase were shared with each other
by a `DS` global. Over time their utility has shrunk and today they no longer align with
the direction of [error management](https://github.com/emberjs/rfcs/blob/master/text/0465-record-data-errors.md) or [type constraints](https://github.com/emberjs/rfcs/pull/740).

## Detailed design

Users would receive a build-time deprecation when importing these methods using the paths
specified in [RFC#395 packages](https://github.com/emberjs/rfcs/blob/master/text/0395-ember-data-packages.md).

They would receive a run-time deprecation when using these methods via the `DS` global.

The deprecation would target `5.0` and would become `enabled` no-sooner than `4.1` (although
it may be made `available` before then).

Users making use of these methods can trivially copy them into their own codebase to continue
using them, though we recommend refactoring to a more direct conversion into the expected errors
format. For refactoring `normalizeModelName` we also recommend following [RFC#740 Deprecate Non-Strict Types](https://github.com/emberjs/rfcs/pull/740).

## How we teach this

Generally usage has not been widely observed and these are not methods commonly shown in 
examples or docs. We should make sure to audit for usages and remove them if they exist.

## Drawbacks

A trivial amount of churn for users that did utilize them.

## Alternatives

Leave them to waste away.
