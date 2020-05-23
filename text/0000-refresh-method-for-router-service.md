- Start Date: 2020-05-23
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# RouterService#refresh

## Summary

> Add a refresh method to the router service that calls refresh on the route specified by `currentRouteName`.

## Motivation

> We want to be able to call refresh the route specified by `currentRouteName` without relying on the send api, which we would like to deprecate. This enables us to get the latest data from the model hook.

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
 * Refreshes the current route, updating it with new data from the model hook.
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
