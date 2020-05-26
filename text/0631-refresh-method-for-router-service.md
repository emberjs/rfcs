- Start Date: 2020-05-23
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/631
- Tracking: (leave this empty)

# RouterService#refresh

## Summary

> Add a refresh method to the router service that calls refresh on all currently active routes.

## Motivation

> We want to be able to call refresh on all currently active routes from a centralized service
or from a component. As a side benefit, we will be able to do this without relying on the send api,
which we would like to deprecate. This enables us to get the latest data from the model hook.

## Detailed design

```js
class RouterService {
    refresh() {
        this._router._routerMicroLib.refresh();
    }
}
```

## How we teach this

> The following documentation will be added to the method:

```js
/**
 * Refreshes all currently active routes, doing a full transition.
 * All resetController, beforeModel, model, afterModel, redirect, and setupController
 * hooks will be called again. You will get new data from the model hook.
 * 
 * @method refresh
 * @public
 */
```

## Drawbacks

> This is a slight increase in API surface area.

## Alternatives

> We could provide a direct link to the current route via the router service. However,
this would encourage people to use routes to store information and provide methods
that should be idiomatically placed in a service.
