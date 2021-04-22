---
Stage: Accepted
Start Date: 2021-03-21
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/733
---

# Add Public `intermediateTransitionTo` Method on `RouterService`

## Summary

This RFC proposes to add a public `intermediateTransitionTo` method on `RouterService` as an equivalent of [`Route#intermediateTransitionTo`](https://api.emberjs.com/ember/3.25/classes/Route/methods/intermediateTransitionTo?anchor=intermediateTransitionTo).

## Motivation

In Ember v3.26 the [`transitionTo`](https://api.emberjs.com/ember/3.25/classes/Route/methods/transitionTo?anchor=transitionTo) / [`replaceWith`](https://api.emberjs.com/ember/3.25/classes/Route/methods/replaceWith?anchor=replaceWith) methods on `Route` and the [`transitionToRoute`](https://api.emberjs.com/ember/3.25/classes/Controller/methods/transitionToRoute?anchor=transitionToRoute) / [`replaceRoute`](https://api.emberjs.com/ember/3.25/classes/Controller/methods/replaceRoute?anchor=replaceRoute) methods on `Controller` will officially be deprecated ([RFC 674](https://github.com/emberjs/rfcs/pull/674)). Once these are removed (in Ember v4), the `intermediateTransitionTo` method on `Route` will be the only public non `RouterService` method available that allows you to transition from one route to another.

The main motivation for this RFC is that, adding an `intermediateTransitionTo` method on `RouterService` would increase consistency as all public transition methods would be available on `RouterService`. This could also be beneficial when teaching Ember to developers who are new to the framework.

This would also allow us to deprecate the `intermediateTransitionTo` method on `Route` later on, making `RouterService` the single point of entry for performing transitions programmatically. Though, this would require a separate RFC.

## Detailed Design

An `intermediateTransitionTo` method will be added on `RouterService` that will function exactly the same as the `intermediateTransitionTo` method on `Route`.

Its signature will look like this:

```ts
/**
   Perform a synchronous transition into another route without attempting
   to resolve promises, update the URL, or abort any currently active
   asynchronous transitions (i.e. regular transitions caused by
   `transitionTo` or URL changes).
   
   This method is handy for performing intermediate transitions on the
   way to a final destination route, and is called internally by the
   default implementations of the `error` and `loading` handlers.
   
   @method intermediateTransitionTo
   @param {String} routeName The name of the route.
   @param {...Object} models The model(s) to be used while transitioning to the route.
   @public
 */
intermediateTransitionTo(routeName: string, ...models: any[]) {}
```

Since the performed transition is synchronous, `intermediateTransitionTo` will _not_ return a [`Transition`](https://api.emberjs.com/ember/3.25/classes/Transition) object.

### Usage

The `intermediateTransitionTo` method on `RouterService` could be used to transition to a `not-found` route when a route's `model` hook returns a `NotFoundError` error. The example below displays how this could be handled globally in the `application` route.

```js
// app/routes/application.js

import { action } from '@ember/object';
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';
import { NotFoundError } from '@ember-data/adapter/error';

export default class ApplicationRoute extends Route {
  @service router;

  @action
  error(error) {
    if (error instanceof NotFoundError) {
      this.router.intermediateTransitionTo('not-found');
    }
  }
}
```

## How We Teach This

At the moment, `intermediateTransitionTo` is not mentioned in the Ember guides as the use for it is so specific. I think we can keep it this way and only add it to the API docs of `RouterService`.

## Drawbacks

- Additional work to be done, though I'm happy to try and do the implementation myself
- A follow-up RFC would be desired to deprecate `intermediateTransitionTo` on `Route` as having multiple public methods to perform the same action is not desired

## Alternatives

- Not add `intermediateTransitionTo` on `RouterService` and keep using `intermediateTransitionTo` on `Route`
- Deprecate `intermediateTransitionTo` on `Route` right away (would require a different RFC)
  - Based on discussions on Discord, it seems that `intermediateTransitionTo` isn't well known ([Discussion 1](https://discord.com/channels/480462759797063690/500803406676492298/801484053844983809) - [Discussion 2](https://discord.com/channels/480462759797063690/500803406676492298/801886820244783113))
  - Others mentioned they'd want to see `intermediateTransitionTo` removed instead ([Discussion](https://discord.com/channels/480462759797063690/500803406676492298/826068398500479036))

## Unresolved Questions

- None at the moment