- Start Date: 2015-09-24
- RFC PR:
- Ember Issue:

# Summary

This RFC proposes:

 - creating a public `router` service that is a superset of today's `Ember.Router`.
 
 - codifying and expanding the supported public API for the `transition` object that is currently passed to `Route` hooks.

These topics are closely related because they share a unified `RouteInfo` type, which will be described in detail.

# Motivation

Given the modern Ember concepts of Components and Services, it is clear that routing capability should be exposed as a Service. I hope this is uncontroversial, given that we already implement it as a service internally, and given that usage of these nominally-private APIs is already becoming widespread.

The immediate benefit of having a `RouterService` is that you can inject it into components, giving them a friendly way to initiate transitions and ask questions about the current global router state.

A second benefit is that we have the opportunity to add new capabilities to the `RouterService` to replace several common patterns in the wild that dive into private internals in order to get things done. There are several places where we leak internals from router.js, and we can plug those leaks.

# Detailed design

## RouterService

By way of a simple example, the router service behaves like this:

```js
import Component from 'ember-component';
import service from 'ember-service/inject';

export default Component.extend({
  router: service(),
  actions: {
    goToMars() {
      this.get('router').transitionTo('planet.mars');
    }
  }
});
```

Like any Service, it can also be injected into Helpers, Routes, etc.

### Relationship between EmberRouter and RouterService

Q: "Why are you calling this thing 'router' when we already have a router? Shouldn't the new thing be called 'routing' or something else?".

A: We shouldn't have two things. From the user's perspective, there is just "the router", and it happens to be available as a service. While we're free to continue implementing it as multiple classes under the hood, the public API should present as a single, coherent concept.

Terminology:

 - `EmberRouter` is the class that we already have today, defined in `ember-routing/system/router` and available publicly as `Ember.Router`
 - `RouterService` is the new class I am proposing.

`EmberRouter` has the following public API today:

 - `map`
 - `location`
 - `rootURL`
 - `willTransition`
 - `didTransition`

That API will be carried over verbatim to `RouterService`, and the publicly accessible `Ember.Router` class will *become* `RouterService`. In terms of implementation, I expect the existing `EmberRouter` class will continue to exist mostly unchanged. But public access to it will be moderated through `RouterService`.

### New Methods: Initiating Transitions

```js
transitionTo(routeName, ...models, queryParams)
replaceWith(routeName, ...models, queryParms)
```

These two have the same semantics as the existing methods on `Ember.Route`:

### New Method: Checking For Active Route

 - `isActive(routeName, ...models, queryParams)`

The arguments have the same semantics as `transitionTo`, the return value is a boolean. This should provide the same logic that determines whether to put an active class on a `link-to`. Here's an example of how we can implement `is-active` as a helper, using this method:

```js
import Helper from 'ember-helper';
import service from 'ember-service/inject';
import observer from 'ember-metal/observer';

export default Helper.extend({
  router: service(),
  compute([routeName, ...models], hash) {
    let allModels;
    if (hash.models) {
      allModels = models.concat(hash.models);
    } else {
      allModels = models;
    }
    return this.get('router').isActive(routeName, ...models, hash.queryParams);
  },
  observer('router.currentRoute', function() {
    this.recompute();
  });
});
```

```hbs
{{!- Example usage -}}
<li class={{if (is-active "person.detail" model) 'chosen'}} >

{{!- Example usage with generic routeName and list of models (avoids splat) -}}
<a class={{if (is-active routeName models=models) 'chosen'}} >

{{!- Note that the complexities of currentWhen can be avoided by composing instead. }}
<a class={{if (or (is-active 'one') (is-active 'two')) 'active'}} href={{url-for 'two'}} >

```


### New Method: URL generation

`url(routeName, ...models, queryParams)`

This takes the same arguments as `transitionTo`, but instead of initiating the transition it returns the resulting URL as a string.

A `url-for` helper can be implemented almost identically to the `is-active` example above.


### New Properties

`currentRoute`: an observable property. It is guaranteed to change whenever a route transition happens (even when that transition only changes parameters and doesn't change the active route). You should consider its value deeply immutable -- we will replace the whole structure whenever it changes. The value of `currentRoute` is a `RouteInfo` representing the current leaf route. `RouteInfo` is described below.

`currentRouteName`:  a convenient alias for `currentRoute.name`.

`currentURL`: provides the serialized string representing `currentRoute`.


### Deprecation

I propose deprecating the publicly extensible `willTransition` and `didTransition` hooks. They are redundant with an observable `currentRoute`, and the arguments they receive leak internal implemetation from router.js.

## RouteInfo Type

A RouteInfo object has the following properties. They are all read-only.

 - name: the dot-separated, fully-qualified name of this route, like `"people.index"`.
 - localName: the final part of the `name`, like `"index"`.
 - attrs: the attributes provided to this route's component, if any, like `{ model }`. For old-style routes that have a controller/view/template instead of a routable component, the implicit `attrs` shall contain `{ model, controller, view }`.
 - params: the values of this route's parameters. Same as the argument to `Route`'s `model` hook. Contains only the parameters valid for this route, if any (params for parent or child routes are not merged).
 - queryParams: the values of any queryParams on this route.
 - parent: another RouteInfo instance, describing this route's parent route, if any.
 - child: another RouteInfo instance, describing this route's active child route, if any.

Notice that the `parent` and `child` properties cause `RouteInfos` to form a linked list. So even though the `currentRoute` property on `RouterService` points at the leafmost route, it can be traversed to discover everything about all active routes. As a convenience, `RouteInfo` also implements `Enumerable` over all the reachable `RouteInfos` from topmost to leafmost. This makes it possible to say things like:

```js
router.currentRoute.find(info => info.name === 'people').params
```

## Transition Object

A `transition` argument is passed to `Route`'s `beforeModel`, `model`, `afterModel`, and `willTransition` hooks. Today it's public API is only really `abort()` and `retry()`.

### New Properties: `from` and `to`

I'm proposing we add `from` and `to` properties on `transition` whose values are `RouteInfo` instances representing the initial and final leafmost routes for this transition. Like all RouteInfos, these are read-only and internally immutable. They are not observable, because a  `transition` instance is never changed after creation.

On an initial full-page load, the `from` property will be `null`. This creates a public API for distinguishing in-app transitions from full-page reloads.

### Example: testing whether route will remain active

Here's an example showing how `willTransition` can figure out if the current route will remain active after the transition:

```js
willTransition(transition) {
  if (!this.transition.to.find(route => route.name === this.routeName)) {
    alert("Please save or cancel your changes.");
    transition.abort();
  }
}
```

### Example: parent redirecting to a fallback model

Here's an example of a parent route that can redirect to a fallback model, without losing its child route:

```js
this.route('person', { path: '/person/:person_id' }, function() {
  this.route('index');
  this.route('detail');
});

//PersonRoute
const fallbackPersonId = 0;
model({ personId }, transition) {
  return this.get('store').find('person', personId).catch(err => {
    this.replaceWith(transition.to.name, fallbackPersonId);
  });
}

// If personId 5 is invalid, and the user visits /person/5/detail, they will get
// redirected to /person/0/detail. And /person/5 will get redirected to /person/0.
```


### Actively discourage use of private API

This RFC provides public API for doing the things people have become accustomed to doing via private properties on `transition`. To eliminate confusion over the correct way, we should hide all the private API away behind symbols, and provide deprecation warnings per our usual release policy around breaking "widely-used private APIs"


# Drawbacks

This RFC suggests only two small deprecations that are unlikely to effect many apps, so the API-churn burden may appear low. However, we know that use of the private APIs we're deliberately disabling is widespread, so users will experience churn. We can provide our usual deprecation cycle to give them early warning, but it still imposes some cost.

This RFC doesn't attempt to change the existing and fairly rich semantics for initiating transitions. For example, you can pass either models or IDs, and those have subtle semantic differences. I think an ideal rewrite would also change the semantics of the route hooks and transitionTo to simplify that area.

# Alternatives

## Less Churn

We could adopt some of the existing broadly used APIs as de-facto public. This avoids churn, but imposes a complexity burden on every new learner, who needs to be told "this is a weird API, but it's what we're stuck with".

## Semver Lawyering

I'm interepreting router.js's public/private documentation as out-of-scope for Ember's semver. The fact that we pass an instance of router.js's Transition as our `transition` argument is not documented. An alternative interpretation is that we need to continue supporting those methods marked as public in router.js's docs.

## Optional Helpers

I didn't propose shipping `is-active` and `url-for` template helpers -- I merely showed that they're easy to build using the router service. But we should arguably just ship them as part of the framework too.

## Branching Route Hierarchies

I am implicitly assuming we will only ever have linear route hierarchies, where a given route has at most one child. I can imagine eventually wanting a way to support branching route hierarchies, where each branch can transition independently. I'm not trying to account for that future.

## Route.parentRoute

This RFC makes it possible for a route to determine its parent's name dynamically via public API, and thus access its parent's model/params/controller:

```js
beforeModel(transition) {
  const parentInfo = transition.to.find(info => info.name === this.routeName).parent;
  const parentModel = this.modelFor(parentInfo.name);
}
```

However, this pattern feels awkward, and I think it justifies just adding a public `parentRouteName()` method to `Route` that would simplify to:

```js
beforeModel(transition) {
  const parentModel = this.modelFor(this.parentRouteName());
}
```
Possibly we *want* this to feel awkward because it's a weird thing to do.



