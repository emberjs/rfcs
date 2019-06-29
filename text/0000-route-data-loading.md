- Start Date: 2019-07-01
- Relevant Team(s): Ember.js
- RFC PR: 
- Tracking: 

# Route Data Loading for Components

## Summary

#499 proposes an API for rendering a component from a route. This RFC builds on it by proposing an API to pass multiple arguments to a component from a route. It's based off the [design proposed by @chadhietala](https://gist.github.com/chadhietala/50b977a7d3476069892d351c65af418c) and much of the text is lifted from there.

## Motivation

The `model()` hook on routes is confusing, because it implies that only one piece of data will be fetched, and it uses the term "model" that is more typically associated with backend development. We should have a better name, and an API that allows data to be passed from a route into a component as multiple named arguments.

## Detailed design

This RFC proposes adding a `load()` hook to `Route`. The route is in the 'loading' state while this hook is run. The keys of the object returned from `load()` become the named arguments to the component.

```js
// routes/profile.js
export default class extends Route {
  @service store;
  async load({ profile_id }) {
    return {
      profile: await this.store.findRecord('profile', profile_id),
    }
  }
}
```

```hbs
{{! components/profile.hbs }}

<h1>{{@profile.firstName}} {{@profile.lastName}}</h1>
```

The `load()` hook can also take responsibility for redirection:

```js
// routes/profile.js
export default class extends Route {
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

Should #499 be accepted, this is a fairly straightforward and complementary change that would reduce confusion from the `model()` hook passing in a `@model` argument to a component. The Guides would need to be updated to use `load()` instead of `model()`, `beforeModel()`, and `afterModel()`.

## Drawbacks

In a time where a lot is changing in the Ember ecosystem, one might argue that this is too much change happening at once.

## Alternatives

* Stick with the `model()` and associated hooks.
* Add `beforeLoad()` and `afterLoad()` hooks as are currently used in conjunction with `model()`.
