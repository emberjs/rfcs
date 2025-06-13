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
  accepted: https://github.com/emberjs/rfcs/pull/1111
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

# Deprecating the `Evented` Mixin

## Summary

Deprecate the `Evented` Mixin in favor of just using the methods from `@ember/object/events`.

## Motivation

For a while now, Ember has not recommended the use of Mixins. In order to fully
deprecate Mixins, we need to deprecate all existing Mixins of which `Evented` is one.

## Transition Path

The `Evented` Mixin provides the following methods: `on`, `one`, `off`, `trigger`, and `has`.

Of these all but `has` are just wrapping methods provided by `@ember/object/events` and
migration is straight-forward:

* `obj.on(name, target, method?)` -> `addListener(obj, name, target, method)`
* `obj.one(name, target, method?)` -> `addListener(obj, name, target, method, true)`
* `obj.trigger(name, ...args)` -> `sendEvent(obj, name, args)`
* `obj.off(name, target, method?)` -> `removeListener(obj, name, target, method)`

Unfortunately, `hasListeners` is not directly exposed by `@ember/object/events`.
We should consider exposing it as the others. In this case, the transition would be:

* `obj.has(name)` -> `hasListeners(obj, name)`

## How We Teach This

In general, I think we should discourage the use of Evented and event handlers as
a core Ember feature and remove most references from the guids.

## Drawbacks

Many users probably rely on this functionality. However, it's almost certainly
something that we don't need to keep in Ember itself.

## Alternatives

* Convert `Evented` to a class decorator-style mixin.

## Unresolved questions

Should we inline the functionality of `Evented` into the classes that use it or are
we also deprecating the methods on those classes?
