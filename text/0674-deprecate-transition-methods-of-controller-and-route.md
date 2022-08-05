---
start-date: 2020-10-12T00:00:00.000Z
release-date:
release-versions: 
  ember-source: v3.26.0

teams: 
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/674
project-link: 
stage: recommended
---

# Deprecate transition methods of controller and route

## Summary

The methods `transitionTo` and `replaceWith` of the `Route` and the methods
`transitionToRoute` and `replaceRoute` of `Controller` should be deprecated.
Existing methods `transitionTo` and `replaceWith` of `RouterService` should
be used instead.

## Motivation

The main motivation is to reduce public API surface related to routing. This
will make the required refactoring of router easier as soon as the methods
have been removed.

The router is known to be poorly documented and underspecified. Especially timing
of the different APIs has been an issue in the past. Some timing related bugs
are open since years.¹

The implementation of `Router#transitionTo`, `Router#replaceWith`,
`Controller#transitionToRoute` and `Controller#replaceRoute` on one side and
`RouterService#transitionTo` and `RouterService#replaceWith` on the other side
are very different. Therefore it's very likely that they have different timings
in edge cases - even if that's not documented. Changing the timing of the
methods is considered a breaking change, which makes refactorings difficult.

Supporting different ways to do the same increases complexity without providing
much value. Especially if it's not only about true shortcuts. Supporting
transitions through `RouterService` and `<LinkTo>` component *only* will reduce
complexity for maintainers.

Additionally it reduces learning costs as new developers do not need to learn
different APIs to do the same thing. The [current guides](https://guides.emberjs.com/release/routing/defining-your-routes/#toc_transitioning-between-routes)
showcase this issue by listing all four methods existing today and when they
could be used:

> It depends on where the transition needs to take place:
>
> - From a template, use <LinkTo /> as mentioned above
> - From a route, use the transitionTo() method
> - From a controller, use the transitionToRoute() method
> - From anywhere else in your application, such as a component, inject the Router Service and use the transitionTo() method

Deprecating `Route#transitionTo` and `Controller#transitionToRoute` will reduce
this list to two options without dropping any use case.

Naming `RouterService` as the only option to trigger a transition in JavaScript
will also help with teaching Ember as a a component-service framework.
`RouterService#transitionTo` and `RouterService#replaceWith` are not only
available in `Route` and `Controller` but also in any services and components.

The current architecture might mislead developers not being aware of
`RouterService` to bubble an action up to the controller or route in order to
trigger a transition. Or pass `Controller#transitionToRoute` or
`Controller#replaceRoute` methods down to a component as an argument.

## Transition Path

`Router#transitionTo`, `Route.replaceWith`, `Controller.transitionToRoute`
and `Controller.replaceRoute` should trigger a deprecation if being used.
The deprecation message should recommend using `RouterService#transitionTo`
or `RouterService#replaceWith` instead.

```diff
  // app/route/foo.js

  import Route from '@ember/routing/route';
  import { inject as service } from '@ember/service';

  export default class FooRoute extends Route {
+   @service router;
    @service session;

    beforeModel() {
      if (!this.session.isAuthenticated) {
-       this.transitionTo('login');
+       this.router.transitionTo('login');
      }
    }
  }
```

```diff
  // app/controllers/foo.js

  import Controller from '@ember/controller';
+ import { inject as service } from '@ember/service';

  export default class FooController extends Controller {
+   @service router;
+
    @action
    async save({ title, text }) {
      let post = this.store.createRecord('post', { title, text });
      await post.save();
-     return this.transitionToRoute('post', post.id);
+     return this.router.transitionTo('post', post.id);
    }
  }
```

A codemod should be provided, which replaces the usage of `transitionTo` method
in any class that extends `Route` and `transitionToRoute` method in any class
that extends `Controller` with `this.router.transitionTo`. The same should be
done for `Route#replaceWith` and `Controller#replaceRoute`. Additionally the
`RouterService` should be injected if needed.

In case consumers face timing issues while refactoring to
`RouterService#transitionTo` and `RouterService#replaceWith` they should change
their project to be compatible with the timing provided by these two methods.
The deprecation phase should be long enough to allow all applications to adopt.
For now it's assumed that it will be enough time for all relevant projects to
migrate before *v5.0* release. The deprecation period might be increased if a
relevant number of projects weren't able to migrate before that date.

## How We Teach This

The guides need to be updated to only use `RouterService#transitionTo`,
`RouterService#replaceWith` and `<LinkTo>` component in order to trigger
a transition.

The routing guides list available option to transition between routes in
[*defining your routes* chapter](https://guides.emberjs.com/release/routing/defining-your-routes/#toc_transitioning-between-routes).
As already discussed in motivation section `Route#transitionTo` and
`Controller#transitionToRoute` should be removed from that list.

The [*redirecting* chapter](https://guides.emberjs.com/release/routing/redirection/)
of routing guides names `Route#transitionTo`, `Controller#transitionToRoute`
and `Route#replaceWith` as options to trigger a redirect. It doesn't mention
`RouterService` yet at all. It also doesn't list `Controller#replaceRoute` as
an option. This two paragraphs should be simplified by naming
`RouterService#transitionTo` and `RouterService#replaceWith` only.

The same applies to [*transitionTo* paragraph of *query parameters* chapter](https://guides.emberjs.com/release/routing/query-params/#toc_transitionto)
of routing guides. Instead of `Route#transitionTo` and
`Controller#transitionToRoute` only `RouterService#transitionTo` should be
mentioned.

Additionally `Route.transitionTo` is used in some code examples in the guides.
These should be changed to use `RouterService#transitionTo` instead.

The deprecated methods should be marked as such in the API docs.

No changes to the tutorial are needed. Transitions in the tutorial are done
using `<LinkTo>` component only.

## Drawbacks

`Route#transitionTo` is a highly used API in existing applications. Deprecating
it will very likely require changes to nearly all existing applications.

Using `RouterService` instead of methods directly available on `Route` or
`Controller` requires explicit injection of router service. This could be seen
as boilerplate code.

Some editors are able to suggest methods which are available on a class
directly which is the case for `Router#transitionTo` and the other methods
proposed for deprecation. This makes discovering them very easy. Discovering
methods only being available on services that need to be injected first, is not
that easy. Developers not being familar with the API might need to reach out
to guides or API docs to find them.

Deprecating (and later removing) these four methods will not enforce all
transitions to be triggered by either `RouterService#transitionTo`,
`RouterService#replaceWith` or `<LinkTo>`. A transition could be still
triggered by setting a controller property, which is bound to a query
parameter. But this should be addressed in a separate RFC, which reworks
registration of query parameters in general.

Ember Engines [injects `Route#transitionToExternal`, `Route#replaceWithExternal`](https://github.com/ember-engines/ember-engines/blob/v0.8.7/packages/ember-engines/addon/-private/route-ext.js)
and [`Controller#transitionToExternalRoute`](https://github.com/ember-engines/ember-engines/blob/v0.8.7/packages/ember-engines/addon/-private/controller-ext.js)
methods. These methods allow the consumer to transition between routes external
to the engine. Ember Engines [does not provide a service like `RouterService` yet](https://github.com/ember-engines/ember-engines/issues/587)
to do this. But it's [under active development](https://github.com/ember-engines/ember-engines/pull/669).

Ember Engines uses the methods deprecated by this RFC to implement
`Route#transitionToExternal`, `Route#replaceWithExternal` and
`Controller#transitionToExternalRoute`. This will trigger a deprecation warning
for all users of Ember Engines until Ember Engines is refactored to not use the
deprecated methods anymore.

While this RFC does not intend to decision how Ember Engines should address
the deprecations, it will very like force Ember Engines to deprecate
`Route#transitionToExternal`, `Route#replaceWithExternal` and
`Controller#transitionToExternalRoute` de facto in mid-term for two reasons:

1. These methods would not align anymore with the method provided by Ember to
   transition between routes.
2. Ember Engines would not be able to change the implementations of these
   methods to not use the methods deprecated by this RFC anymore. Using the
   `RouterService` would very likely change timing and other details of the
   methods, which could be seen as breaking changes. Keeping the current
   timings and other details would require usage of private method of the
   router.

## Alternatives

Three possible alternative are discovered so far:

1. Instead of deprecating `Route#transitionTo`, `Route#replaceWith`,
   `Controller#transitionToRoute` and `Controller#replaceWith` we could try to
   align their implementations to match `RouterService#transitionTo` and
   `RouterService.replaceWith` in regards to timing and other details.

   Doing so will very likely require breaking changes as existing applications
   do very likely depend onto the existing timings and other details. Even if
   not being specified and documented these details became part of our public
   API over the years.

   Therefore we would need to introduce an optional feature which allows
   applications to opt-in into the new timings as soon as they have verifed
   everything is working as expected.

2. We could introduce new methods on `Route` and `Controller`, which are true
   shortcuts for using `RouterService`. This would prevent introducing the need
   to explicitly inject `RouterService` in `Route` and `Controller` in order to
   trigger a transition. As mentioned above this could be seen as boilerplate.

   Coming up with good names for these new methods would be challening as most
   of the names has been taken already. Additionally introducing new methods on
   `Controller` and `Route` will not help with reducing public API surface and
   teaching a component-service architecture.

3. We could do nothing and continue the different ways to trigger a transition
   in JavaScript - including different timings and other details.

## Unresolved questions

No open questions have been discovered so far.

## Footnotes

1. Some example for routing related bugs which are open for quite some time
   now:

   - https://github.com/emberjs/ember.js/issues/10262
   - https://github.com/emberjs/ember.js/issues/11152
   - https://github.com/emberjs/ember.js/issues/12945
   - https://github.com/emberjs/ember.js/issues/14875
   - https://github.com/emberjs/ember.js/issues/15801
   - https://github.com/emberjs/ember.js/issues/18416
   - https://github.com/emberjs/ember.js/issues/18577
   - https://github.com/emberjs/ember.js/issues/19037

   This list does not contain a represantive list of bugs. Neither does it
   include bugs, which I verified myself. It's nothing more than a collection
   of bug reports that I found by [searching for issues containing
   *transitionTo* in ember.js repository](https://github.com/emberjs/ember.js/search?p=1&q=transitionTo&type=issues).
