---
Start Date: 2020-05-23
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/631
Tracking: (leave this empty)

---

# RouterService#refresh

## Summary

Add a refresh method to the router service that calls refresh on all currently active routes,
or refreshes the descendents of the active route referred to by the pivot route name provided as an argument.

## Motivation

We want to be able to call refresh on all currently active routes, or a subset of them,
from a centralized service or from a component.
As a side benefit, we will be able to do this without relying on the send api,
which is being discussed as a possible deprecation in
[RFC 632](https://github.com/emberjs/rfcs/pull/632).
This enables us to get the latest data from the model hook.

## Detailed design

The following pseudocode represents the overall technical design of this method.
This code implementation is not normative.

```js
class RouterService {
    refresh(pivotRouteName?: string): Transition {
        let pivotRoute = pivotRouteName && lookupRoute(pivotRouteName);
        assert("If an argument provided must be the name of an active route", !pivotRouteName || isActiveRoute(pivotRoute));
        return this._router._routerMicroLib.refresh(pivotRoute);
    }
}
```

where *lookupRoute* gets the route specified by the pivotRouteName,
and *isActiveRoute* determines if the specified route is active.
The method optionally takes the route name that will, along with its descendents, be refreshed.
The method will return a promise that resolves when the refresh is complete.

## How we teach this

The following documentation will be added to the method:

```js
/**
 * Refreshes all currently active routes, doing a full transition.
 * If a pivotRouteName is provided and refers to a currently active route,
 * it will refresh only that route and its descendents.
 * Returns a promise that will be resolved once the refresh is complete.
 * All resetController, beforeModel, model, afterModel, redirect, and setupController
 * hooks will be called again. You will get new data from the model hook.
 * 
 * @method refresh
 * @param {String} [pivotRouteName] the route to refresh (along with all child routes)
 * @return Transition
 * @public
 */
```

## Drawbacks

This is a slight increase in API surface area.

## Alternatives

We could provide a direct link to the current route via the router service. However,
this would encourage people to use routes to store information and provide methods
that should be idiomatically placed in a service.
