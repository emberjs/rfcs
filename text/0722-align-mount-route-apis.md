---
Stage: Accepted
Start Date: 2021-02-12
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/722
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

# Align mount and route APIs

## Summary

Update the Ember `mount` API to accept a function argument which defines the Engine’s routes similar to how the `route` API works today.

## Motivation

Some Ember applications utilize in-repo ember-engines strictly for their lazy loading and code organization features and not for their isolation features. In this context the isolation actually causes some maintenance overhead and prevents build optimization.

If isolation is not a requirement and all engines in an application are in-repo it makes more sense to define all of the routes at the application level in the router.js.

## Detailed design

The required change will be made to the `mount` method in [packages/@ember/-internals/routing/lib/system/dsl.ts ](https://github.com/emberjs/ember.js/blob/b35106e4d8471e396eb1ee5a5044b6bbe72fa069/packages/%40ember/-internals/routing/lib/system/dsl.ts#L176). A third argument will be added matching the method signature of the `route` method.

```js
mount(name, options, mountCallback) {...}
```

If the argument is present and it is a function the route-map will not be resolved and the callback will be used in place of the resolved route-map.

```js
let engineRouteMap = null;

if (!mountCallback) {
  engineRouteMap = this.options.resolveRouteMap(name);
}

if (mountCallback) {
  mountCallback.call(childDSL);
} else if (engineRouteMap) {
  engineRouteMap.class.call(childDSL);
}
```

## How we teach this

The alignment of the `route` and `mount` APIs should be easy to teach and is likely already inline with peoples mental model of the `route` API.

This change also unlocks a more idomatic way of using in-repo engines when the use-case does not require isolation.

The Ember-Engines documentation will need to be updated to include instructions for defining engine routes via the mount API.

## Drawbacks

One drawback is that this changes an existing public API and is something that existing Ember-Engines users may need to learn.

## Alternatives

Create an addon which extends the resolver to resolve route-maps from a custom location.
