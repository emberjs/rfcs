- Start Date: 2019-07-01
- Relevant Team(s): Ember.js
- RFC PR: 
- Tracking: 

# Route Data Loading for Components

## Summary

[#731 Add setRouteComponent API](https://github.com/emberjs/rfcs/pull/731) proposes an API for rendering a component from a route. This RFC builds on it by proposing an API to pass multiple arguments to a component from a route. It's based off the [design proposed by @chadhietala](https://gist.github.com/chadhietala/50b977a7d3476069892d351c65af418c) and much of the text is lifted from there.

## Motivation

The `model()` hook on routes is confusing, because it implies that only one piece of data will be fetched, and it uses the term "model" that is more typically associated with backend development. We should have a better name, and an API that allows data to be passed from a route into a component as multiple named arguments.

## Detailed design

This RFC proposes adding a `load()` hook to `Route`. 

The `load()` hook will receive the same arguments as `model()`.

The route is in the `loading` state while this hook is run. When this hook has finished running, the route will change to the `resolved` state (and the `setRouteComponent` component will be rendered).

The keys of the object (what `Object.keys()` would return, i.e. only the own (`hasOwnProperty`) props, and nothing from prototypes) returned from `load()` become the named arguments to the component.

```js
// routes/profile.js
import { hash as resolveValues } from 'rsvp';
export default class ProfileRoute extends Route {
  @service store;
  async load({ profile_id }) {
    return await resolveValues({
      profile: this.store.findRecord('profile', profile_id),
      comments: this.store.query('comments', { profile_id })
    });
  }
}
```

```hbs
{{! components/profile.hbs }}

<h1>{{@profile.firstName}} {{@profile.lastName}}</h1>
```

If something other than an object is returned, an exception is thrown.

If both `model()` and `load()` are present, an exception is thrown.

As suggested in [Always run model hook #283](https://github.com/emberjs/rfcs/pull/283), the `load()` hook will always be run, even if a model is supplied to `{{link-to}}`.

The `load()` hook will not support the "magic" behavior of the model hook, in which it will make loading work magically in simple scenarios without actually implementing the hook (http://api.emberjs.com/ember/release/classes/Route/methods/model?anchor=model).

The `load()` hook can also take responsibility for redirection:

```js
// routes/profile.js
export default class ProfileRoute extends Route {
  @service store;
  @service router;
  @service user
  async load({ profile_id }) {
    if (this.user.loggedIn) {
      return {
        profile: await this.store.findRecord('profile', profile_id),
      };
    } else {
      await this.router.transitionTo('login');
    }
  }
}
```

## How we teach this

Should #731 be accepted, this is a fairly straightforward and complementary change that would reduce confusion from the `model()` hook passing in a `@model` argument to a component. The Guides would need to be updated to use `load()` instead of `model()`, `beforeModel()`, and `afterModel()`.

We could also include the `load()` hook in the route blueprint, returning an empty object, to make the hook more easily discoverable.

## Drawbacks

Given that `setRouteComponent` is intended to unlock experimentation rather than being a final solution, adding this API at the present time could be premature.

## Alternatives

* Stick with the `model()` and associated hooks.
* Add `beforeLoad()` and `afterLoad()` hooks as are currently used in conjunction with `model()`.
