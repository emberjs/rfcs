---
start-date: 2021-05-26T00:00:00.000Z
release-date:
release-versions: 
teams: 
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/750
project-link: 
stage: accepted
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

Now that Ember is dropping support for IE11, we no longer need `Ember.assign` as a polyfill since `Object.assign`
is available in all browsers that Ember v4.x supports ([CanIUse](https://caniuse.com/mdn-javascript_builtins_object_assign), [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign#browser_compatibility)).

## Motivation

The polyfill is no longer necessary, as well as being another Emberism that can be removed. Apps and addons can use `Object.assign` or object destructuring depending on their browser support targets.

## Transition Path

The transition path is relatively simple: apps that use Ember 4.x will replace `Ember.assign` with `Object.assign`, and apps and addons that use or support Ember 3.x can continue to use the polyfill if needed.

ex:
```
import { assign as emberAssign } from '@ember/polyfills';

const assign = Object.assign || emberAssign;
```

## How We Teach This

A descriptive deprecation message alerting a developer that `Ember.assign` is deprecated and can be replaced with `Object.assign`.

## Drawbacks

The only drawback is replacing the polyfill assign with the native assign, but there is minimal effort to do this.

## Alternatives

Doing nothing.

## Unresolved questions

None.
