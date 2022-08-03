---
# FIXME: This may be a further stage
Stage: Accepted
Start Date: 2021-04-23
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): ember-data
RFC PR: https://github.com/emberjs/rfcs/pull/738
---

# EmberData | Deprecate Model Reopen

## Summary

Deprecates using `reopen` to alter clases extending `@ember-data/model`.

## Motivation

`reopen` restricts the ability of EmberData to design better primitives that maintain
compatibility with `@ember-data/model`, and is a footgun that leads to confusing incorrect
states for users that utilize it.

## Detailed design

The static `reopen` method on `Model` will be overwritten to provide a deprecation which
once resolved or after the deprecation lifecycle completes will result in `reopen` throwing
an error when used in dev and silently no-oping in production. This deprecation will target
`5.0` and not be `enabled` sooner than `4.1` though it may come available before that.

## How we teach this

This is best taught through a deprecation guide. Users using `reopen` to test multiple
configurations in their test suite should instead extend and register a new model each time.

Users using `reopen` to modify a class immediately after creating it should also refactor
to `extend` instead.

Users using `reopen` to modify a class dynamically at runtime should refactor to either register
new model types or (better) utilize a megamorphic solution such as ember-m3 to achieve their needs.

In all cases, using `reopen` after a class instance for a record has already been created has *always*
resulted in at least minor and potentially major errors in application state.

## Drawbacks

Test suites, including EmberData's own, that make use of `reopen` are often "order dependent" in order
for the test suite to pass, and refactoring them can sometimes be a difficult exercise in determining
which tests had modified the class to achieve the model shape needed by another test. In general though
the drawbacks here are small given the widespread adoption of class syntax and growing adoption of octane
paradigms.

## Alternatives

- Don't deprecate `reopen` and wait to replace `@ember-data/model` in it's entirety. This alternative would prevent us from providing custom decorators not extending from `computed` and limit potential build tools allowing users to optimize existing usage of EmberData.

- Wait for `ember` to deprecate `reopen`. Because we cannot build custom decorators and support `reopen` without asking for `ember` to make available to use private APIs I do not think we should wait for Ember here. This RFC does not preclude or force Ember to deprecate reopen more broadly.
