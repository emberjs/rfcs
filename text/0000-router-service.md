- Start Date: 2015-09-24
- RFC PR:
- Ember Issue:

# Summary

This RFC proposes:

 - creating a public `router` service that is a superset of today's `Ember.Router`.
 
 - codifying and expanding the supported public API for the `transition` object that is currently passed to `Route` hooks.

 - introducing the concept of dynamically-scoped routing info.

These topics are closely related because they share a unified `RouteInfo` type, which will be described in detail.

# Motivation

Given the modern Ember concepts of Components and Services, it is clear that routing capability should be exposed as a Service. I hope this is uncontroversial, given that we already implement it as a service internally, and given that usage of these nominally-private APIs is already becoming widespread.

The immediate benefit of having a `RouterService` is that you can inject it into components, giving them a friendly way to initiate transitions and ask questions about the current global router state.

A second benefit is that we have the opportunity to add new capabilities to the `RouterService` to replace several common patterns in the wild that dive into private internals in order to get things done. There are several places where we leak internals from router.js, and we can plug those leaks.

A `RouterService` is great for asking global questions, but some questions are not global and today we incur complexity by treating them as if they are. For example:

 - `{{link-to}}` can use implicit models from its context, but that breaks when you're trying to animate to or from a state where those models are not present.

 - `{{link-to}}` has a lot of complexity and performance cost that deals with changing its active state, and the precise timing of when that should happen.

 - there is no way to ask the router what it would do to handle a given URL without actually visiting that URL.

All of the above can be addressed by embracing what is already internally true: "the current route" is not a single global, it's a dynamically-scoped variable that can have different values in different parts of the application simultaneously.

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
replaceWith(routeName, ...models, queryParams)
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
  didTransition: observer('router.currentRoute', function() {
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

`urlFor(routeName, ...models, queryParams)`

This takes the same arguments as `transitionTo`, but instead of initiating the transition it returns the resulting root-relative URL as a string (which will include the application's `rootUrl`).

A `url-for` helper can be implemented almost identically to the `is-active` example above.

### New Method: URL recognition

`recognize(url)`

Takes a string URL and returns a `RouteInfo` for the leafmost route represented by the URL. Returns `null` if the URL is not recognized. This method expects to receive the actual URL as seen by the browser _including_ the app's `rootURL`.


Example: this feature can replace [this use of private API in ember-href-to](https://github.com/intercom/ember-href-to/blob/b8cf0699eec6a65570b05e4fc22b27d8cea49c42/app/instance-initializers/browser/ember-href-to.js#L34).


### New Method: Recognize and load models

`recognizeAndLoad(url)`

Takes a string URL and returns a promise that resolves to a `RouteInfoWithAttributes` for the leafmost route represented by the URL. The promise rejects if the URL is not recognized or an unhandled exception is encountered. This method expects to receive the actual URL as seen by the browser _including_ the app's `rootURL`.

### Deprecating willTransition and didTransition

Application-wide transition monitoring events belong on the Router service, not spread throughout the Route classes. That is the reason for the existing `willTransition` and `didTransition` hooks/events on the Router. But they are not sufficient to capture all the detail people need. See for example, https://github.com/nickiaconis/rfcs/blob/1bd98ec534441a38f62a48599ffa8a63551b785f/text/0000-transition-hooks-events.md

In addition, they receive handlerInfos in their arguments, which are an undocumented internal implementation detail of router.js that doesn't belong in Ember's public API. Everything you can do with handlerInfos can be done with the RouteInfo type that is proposed in this RFC, with the benefit of sticking to supported public API.

So we should deprecate willTransition and didTransition in favor of the following new events.

### New Events: routeWillChange & routeDidChange

The `routeWillChange` event fires whenever a new route is chosen as the desired target of a transition. This includes `transitionTo`, `replaceWith`, all redirection for any reason including error handling, and abort. Aborting implies changing the desired target back to where you already were. Once a transition has completed, `routeDidChange` fires. 

Both events receive a single `transition` argument as described in the "Transition Object" section below, which explains the meaning of `from` and `to` in more detail.

Redirection example:

 1. current route is A
 2. user initiates a transition to B
 3. routeWillChange fires `from` A `to` B.
 4. B redirects to C
 5. routeWillChange fires `from` A `to` C.
 6. routeDidChange fires `from` A `to` C.

Abort example:

 1. current route is A
 2. user initiates a transition to B
 3. routeWillChange fires `from` A `to` B.
 4. in response to the previous routeWillChange event, the transition is aborted.
 5. routeWillChange fires `from` A `to` A.
 8. routeDidChange fires `from` A `to` A.

Error example:

 1. current route is A
 2. user initiates a transition to B.index
 3. routeWillChange fires `from` A `to` B.
 4. B throws an exception, and the router discovers a "B-error" template.
 5. routeWillChange fires `from` A `to` B-error
 6. routeDidChange fires `from` A `to` B-error

These are events, not extension hooks -- now that we are exposing a Service, it makes more sense to subscribe to its events than extend it.

### New Properties

`currentRoute`: an observable property. It is guaranteed to change whenever a route transition happens (even when that transition only changes parameters and doesn't change the active route). You should consider its value deeply immutable -- we will replace the whole structure whenever it changes. The value of `currentRoute` is a `RouteInfo` representing the current leaf route. `RouteInfo` is described below.

`currentRouteName`:  a convenient alias for `currentRoute.name`.

`currentURL`: provides the serialized string representing `currentRoute`.

### Query Parameter Semantics

Today, `queryParams` impose unnecessarily high cost because we cannot generate URLs or determine if a link is active without taking into account the default values of query parameters. Determining their default values is expensive, because it involves instantiating the corresponding controller, even in cases where we will never visit its route.

Therefore, the `queryParams` argument to the new `urlFor`, `transitionTo`, `replaceWith`, and `isActive` methods defined in this document will behave differently.

 - default values will not be stripped from generated URLs. For example, `urlFor('my-route', { sortBy: 'title' })` will always include `?sortBy=title`, whether or not `title` is the default value of `sortBy`.

 - to explicitly unset a query parameter, you can pass the symbol `Ember.DEFAULT_VALUE` as its value. For example, `transitionTo('my-route', { sortBy: Ember.DEFAULT_VALUE })` will result in a URL that does not contain any `?sortBy=`.

(Sticky parameters are still allowed, because they only apply when the destination controller has already been instantiated anyway.)



## RouteInfo Type

A RouteInfo object has the following properties. They are all read-only.

 - name: the dot-separated, fully-qualified name of this route, like `"people.index"`.
 - localName: the final part of the `name`, like `"index"`.
 - params: the values of this route's parameters. Same as the argument to `Route`'s `model` hook. Contains only the parameters valid for this route, if any (params for parent or child routes are not merged).
 - paramNames: ordered list of the names of the params required for this route. It will contain the same strings as `Object.keys(params)`, but here the order is significant. This allows users to correctly pass params into routes programmatically.
 - queryParams: the values of any queryParams on this route.
 - parent: another RouteInfo instance, describing this route's parent route, if any.
 - child: another RouteInfo instance, describing this route's active child route, if any.

Notice that the `parent` and `child` properties cause `RouteInfos` to form a linked list. So even though the `currentRoute` property on `RouterService` points at the leafmost route, it can be traversed to discover everything about all active routes. As a convenience, `RouteInfo` also implements `Enumerable` over all the reachable `RouteInfos` from topmost to leafmost. This makes it possible to say things like:

```js
router.currentRoute.find(info => info.name === 'people').params
```

## RouteInfoWithAttributes

This type is almost identical to `RouteInfo`, except it has one additional property named `attributes`. The attributes contain the data that was loaded for this route, which is typically just `{ model }`.

## Transition Object

A `transition` argument is passed to `Route#beforeModel`, `Route#model`, `Route#afterModel`, `Route#willTransition`, and `Router#willTransition`. Today `transition`'s public API is only really `abort()` and `retry()`.

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

This RFC provides public API for doing the things people have become accustomed to doing via private API. To eliminate confusion over the correct way, we should hide all the private API away behind symbols, and provide deprecation warnings per our usual release policy around breaking "widely-used private APIs".

Some of the private APIs we should mark and warn include:

 - transition.state
 - transition.params
 - `lookup('router:main')` (should use `service:router` instead)


## Dynamically-Scoped Variables

The next section introduces general-purpose, dynamically-scoped variables. Why am I doing this in "The Router Service RFC"? Because dynamically-scoped RouteInfo has several nice capabilities that will be introduced in the subsequent section. And once we have one dynamically-scoped variable, we have the option of exposing that capability for other uses.

### In the general case

A [dynamically-scoped variable](https://en.wikipedia.org/wiki/Scope_(computer_science)#Dynamic_scoping) takes its value from its calling context, as opposed to its lexical context. Since dynamically-scoped variables are powerful to the point of potential danger, their syntax is intended to be appropriately verbose.

Reading a dynamically-scoped variable in handlebars:

 ```hbs
 {{!- retrieve the value of a dynamically scoped variable }}
 (get-dynamic-variable "myVariableName")
 ```

Defining a component that reads a dynamically-scoped variable:

```js
let MyComponent = Ember.Component.extend({
  didInsertElement() {
    // Access to the variable here is intended to be indistinguishable
    // from a normal, explicitly-passed input argument. 
    doSomethingWith(this.get('myDynamicVariable'));
  }
});
MyComponent.reopenClass({
  // This is where the specialness happens.
  getDynamicVariables: [ 'myDynamicVariable' ]
});
```

Defining a helper that reads a dynamically-scoped variable:

```js
let MyHelper = Ember.Helper.extend({
  compute(params, hash) {
    return doSomethingWith(hash.myDynamicVariable);
  }
});
MyHelper.reopenClass({
  getDynamicVariables: [ 'myDynamicVariable' ]
});
```

Setting a dynamically-scoped variable:

```hbs
{{#with-dynamic-variable "myVariableName" someValue}}
  {{!-
    within this block AND ALL ITS DESCENDANTS until
    otherwise overridden by another with-dynamic-variable statement, 
    `get-dynamic-variable "myVariableName"` returns someValue.
  -}}
{{/with-dynamic-variable}}
```

Note that there is no `set-dynamic-variable`. You can only introduce new scopes, not mutate your containing scope. There is also no way to set a dynamically-scoped variable directly from Javascript -- your component must use a `with-dynamic-variable` block within its handlebars template.

User-defined, dynamically-scoped variables must be pre-declared before first render via `Ember.declareDynamicVariable("myVariableName")`. Attempting to set one that wasn't declared is an error. Attempting to read one that doesn't exist returns `undefined`. Predeclaration is an optimization that allows us to maintain consistent performance within Glimmer.

### The specific case of routeInfo

`routeInfo` is a dynamically-scoped variable provided by Ember as public API. Its value is always a `RouteInfoWithAttributes` object that is correct for the given route context. This enables several nice things, which I will illustrate with examples:

1. Here is a simplified `is-active` helper that will always update at the appropriate time to match exactly what is rendered in the current outlet. It will maintain the correct state even during animations. Instead of injecting the router service, it consumes the `routeInfo` from its containing environment:

```js
Ember.Helper.extend({
  compute([routeName], { routeInfo }) {
    return !!routeInfo.find(info => info.name === routeName);
  }
}).reopenClass({
  getDynamicVariables: ['routeInfo']
});
```

A more complete version that also matches models and queryParams can be written in the same way. 

2. We can improve `link-to` so that it always finds implicit model arguments from the local context, rather than trying to locate them on the global router service. This will fix longstanding bugs like https://github.com/ember-animation/liquid-fire/issues/347 and it will make it easier to test components that contain `{{link-to}}`. This would also open the door to relative link-tos.

3. `liquid-outlet` can be implemented entirely via public API. It would become:

```hbs
{{#liquid-bind (get-dynamic-variable "routeInfo") as |currentRouteInfo|}}
  {{#with-dynamic-variable "routeInfo" currentRouteInfo}}
    {{outlet}}
  {{/with-dynamic-variable}}
{{/liquid-bind}}
```

4. Prerendering of non-current routes becomes possible. You can use `recognizeAndLoad` to obtain a `RouteInfoWithAttributes` and then use `{{#with-dynamic-variable "routeInfo" myRouteInfo}} {{outlet}} {{/with-dynamic-variable}}` to render it.


# Drawbacks

This RFC deprecates only two public extension hooks API, so the API-churn burden may appear low. However, we know that use of the private APIs we're deliberately disabling is widespread, so users will experience churn. We can provide our usual deprecation cycle to give them early warning, but it still imposes some cost.

This RFC doesn't attempt to change the existing and fairly rich semantics for initiating transitions. For example, you can pass either models or IDs, and those have subtle semantic differences. I think an ideal rewrite would also change the semantics of the route hooks and transitionTo to simplify that area.

# Alternatives

## Less Churn

We could adopt some of the existing broadly used APIs as de-facto public. This avoids churn, but imposes a complexity burden on every new learner, who needs to be told "this is a weird API, but it's what we're stuck with".

## Semver Lawyering

I'm interpreting router.js's public/private documentation as out-of-scope for Ember's semver. The fact that we pass an instance of router.js's Transition as our `transition` argument is not documented. An alternative interpretation is that we need to continue supporting those methods marked as public in router.js's docs.

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

## Naming of Ember.DEFAULT_VALUE Symbol

Should we introduce new API via the `Ember` global and switch to a module export once all the rest of Ember does, or should we just start with a module export right now? If so, what module?

    import { DEFAULT_VALUE } from 'ember-routing';

## Generic dynamically-scoped variables may be too powerful

They let you do spooky-action-at-a-distance stuff. We could instead choose to implement single-purpose methods like `(get-route-info)` and `{{#with-route-info someValue}}` that would solve the routing case without opening the door to arbitrary user-defined ones.

