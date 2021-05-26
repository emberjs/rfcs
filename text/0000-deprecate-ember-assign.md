---
Stage: Accepted
Start Date: 2021-05-26
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/750
---

<!---
Directions for above:

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Deprecate Ember.assign

## Summary

Deprecate `Ember.assign` starting in v4.0. Now that Ember is dropping support for IE11, we no longer need `Ember.assign` as a polyfill since `Object.assign`
is available in all browsers that Ember v4.0+ supports ([CanIUse](https://caniuse.com/mdn-javascript_builtins_object_assign), [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign#browser_compatibility)).

## Motivation

The polyfill is no longer necessary and is a small amount of code that can be removed and remove another Emberism. Apps and addons can use `Object.assign` or object destructring depending on their browser support targets.

## Transition Path

The transision path is quite straightforward: devs will replace `Ember.assign` with `Object.assign`.

## How We Teach This

A descriptive deprecation message alerting a developer that as of now (Ember v4), `Ember.assign` is no longer needed and it can be replaced with `Object.assign`.

## Drawbacks

The only drawback is replacing the polyfill assign with the native assign, but there is minimal effort to do this.

## Alternatives

Doing nothing.

## Unresolved questions

None.
