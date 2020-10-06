- Start Date: 2020-10-04
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Implicit Injection Deprecation

## Summary

This RFC seeks to deprecate implicit injections on arbitrary Ember Framework objects. This is commonly done via `owner.inject` in an
intializer or instance-initializer.

A prevalent example of implicit injection is found in `ember-data` [injecting](https://github.com/emberjs/data/blob/4bd2b327c4cbca831f9e9f8bc6b497200a212f9b/packages/-ember-data/addon/setup-container.js) their default `store` into all
Ember.Route and Ember.Controller factory objects. Here is a pruned example of how this looks.

```js
// app/initializers/store-inject.js
export function initialize(application) {
    application.inject('route', 'store', 'service:store');
    application.inject('controller', 'store', 'service:store');
}

export default {
    name: 'store-inject',
    initialize,
};
```

```js
export default class PostRoute extends Route {
  // This proposal seeks to make this service injection explicit
  // Currently users do no need to specify this injection if ember-data is installed
  // @service store;

  model() {
    return this.store.findRecord('post', 1);
  }
}
```

Ensuring a property is present on Ember's Framework objects should be explicitly defined. This will allow the Ember ecosystem to
further progress the framework while easing the learning curve for all developers.

## Motivation

Implicit injections have long confused developers onboarding Ember projects. Without an
explicit injection visible on the class body, developers start to wonder how certain properties got there.
We can infer from a common implicit injection in `ember-data` to reason about some of the downsides this presents.
If a project has `ember-data` installed, an initializer at runtime registers the `@ember-data/store` on both
Ember.Route and Ember.Controller factory objects.  This is a nice convenience for a project with many routes
that are used for data fetching. However, it leads to many inconveniences that hinder learning and advancement
of the framework. This includes:

1. By entangling specific objects with the whole dependency tree, we might be preventing incremental adoption.
   For example, `ember-data` has laid out its plan for adopting only specific packages in [Project Trim](https://github.com/emberjs/data/issues/6166).
   However, because `store` is injected by default on all Ember.Route and Ember.Controller factory objects,
   a user cannot easily opt out of `@ember-data/store`. This is a common problem space in other communities as well.
   With previous JetBrains IDE installations, adding plugins required a reload. However, by removing a contructor function
   that injected all the dependencies at once, they were able to avoid reload after installation of a plugin.

2. Eager initialization of the dependency tree from which the property came from prevents tree shaking and incurs a
   performance hit. For example, if you app does not need `ember-data` to render the entry route, the `store` injection will
   take up more CPU cycles than necessary, hurting common user facing metrics.

3. Eager initialization makes it hard to users to write tests that need to take advantage of a stub.
   This is often a silent and annoying hurdle that users have to get around when writing tests.
   Moreover, this is especially apparent for users moving to Ember's new testing APIs in [RFC 232](https://github.com/emberjs/rfcs/blob/master/text/0232-simplify-qunit-testing-api.md) and [RFC 268](https://github.com/emberjs/rfcs/blob/master/text/0268-acceptance-testing-refactor.md),
   ensuring instance initializers run before each test.

4. Injection on commonly used framework objects can lead to deviation of project specific styleguides. For instance,
   injecting the `router` on the Ember.Component factory object will certainly lead to instances where the `router` is explicitly
   injected and other instances where is it implicitly injected.

5. Users have come to associate `Route.model()` with a hook that returns `ember-data` models in the absense of explicit injection.
   While this can be true, it is not wholly true. New patterns of data loading are becoming
   accepted in the community including opting to fetch data in a Component.


## Detailed design

Removing implicit injections is not possible until we first issue deprecations.
This helps us maintain our strict semver commitment to not completing a major version bump and
removing APIs without a deprecation path. As a result, it will take us 3 phases to remove `owner.inject`.

### 1. Deprecate implicit injection on target object

The first step will be issuing a deprecation on the target factory object that is receiving a property. This will surface the deprecation
on user's owned objects including transitively through addons like `ember-data`. This can be accomplished by installing a native getter
and setter on the target. Whenever the property is accessed, we will check if the property is explicitly injected. If it isn't,
we will issue a deprecation with an until value, id and url to read more about this deprecation.

### 2. Deprecate `owner.inject`

The first phase did not actually deprecate the "use" of `owner.inject`.  As a result, we need to deprecate
it's use directly before removing.

### 3. Profit!

Ember 4.0 will remove the ability to utilize `owner.inject` to inject arbitrary properties on Ember's Framework objects.

It is important to consider the timeline of these three phases.  Many addons and apps will need to make minor and major
changes to their codebase. We need to consider this as we move through each phase.

## How we teach this

The API docs would need to be overhauld in a few spots. First, we need to remove the [docs](https://guides.emberjs.com/release/applications/dependency-injection/#toc_factory-injections) about Factory injections.
Second, we need to detail explicit injection where currently relying on implicit injection.  Many examples show fetching data
via `this.store` but do not specifiy how `this.store` arrived as a property on the Route. Also apparent
disclaimers for people visiting the `ember-data` docs that explicit injection of `store` is necessary
and has deviated from past behaviour is likely prudent.

The default blueprints when generating a new Ember application will not change.

## Alternatives

- Continue with use of `owner.inject` but overhaul docs and recommend explicit injection.
- Provide application level super class with a default store injection provided by `ember-data` and other common examples in the community.
- Allow injection on a specific factory with a single backing Class instead of a generic factory object (e.g. `route` vs `route.index`).

## Open Questions

- Should we simply replace the [docs](https://guides.emberjs.com/release/applications/dependency-injection/#toc_factory-injections) on Factory Injections without an alternative?

## Related links and RFCs
- [Deprecate defaultStore located at `Route.store`](https://github.com/emberjs/rfcs/issues/377)
- [Pre-RFC: Deprecate implicit injections (owner.inject)](https://github.com/emberjs/rfcs/issues/508)
- [Deprecate implicit record loading in routes](https://github.com/emberjs/rfcs/issues/557)
