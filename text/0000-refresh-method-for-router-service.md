- Start Date: 2020-05-23
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# RouterService#refresh

## Summary

> Add a refresh method to the router service that calls refresh on the current route.

## Motivation

> We want to be able to refresh the current route without relying on the send api.

## Detailed design

```
class RouterService {
    refresh() {
        this.owner.lookup(`route:${this.currentRouteName}`).refresh();
    }
}
```

## How we teach this

> Documentation will be added to the method.

## Drawbacks

> This is a slight increase in API surface area.

## Alternatives

> We could provide a direct link to the current route via the router service.
