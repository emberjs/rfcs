- Start Date: 2015-10-01
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Currently, Ember waits until a route's `beforeModel` hook resolves before triggering its `model` hook, waits until its `model` hook resolves before triggering its `afterModel` hook, and waits until its `afterModel` hook resolves before triggering a child route's `beforeModel` hook.
Performance wins can be achieved by running in parallel the model hooks for all routes related to a transition.

# Motivation

There are performance wins associated with running model hooks in parallel.
Child routes that do not depend on data from parent routes may make their requests at the same time as their parent routes.
This causes the child routes to resolve faster.
APIs that batch requests can experience additional performance gains because requests will batch across routes.
See this demo for a visualization: http://nickiaconis.github.io/ember-parallel-model-demo/

# Detailed design

In short, we need to run all `beforeModel`, `model`, and `afterModel` hooks before resolving the promises they return.
This has two major implications:

1. The `afterModel` hook will be passed a promise, since the result of the `model` hook will not have resolved by the time `afterModel` is called.
2. Routes may not have resolved the data requested by `modelFor` when it is called. It will need to return a promise or there will need to be another method for getting a promise that eventually resolves a route's model. (`modelPromiseFor`?)

#### Router.js (tildeio/router.js#171)

`handler-info` presently has a `resolve` method that calls the model hooks. We move the model hook calls to a new `request` method, and save the promises on the `handler-info`'s context (the promise from the `model` hook is passed to the `afterModel` hook). The `resolve` method takes the stored promises and inserts them into the resolution chain.

`transition-state` presently has a `resolveOneHandlerInfo` method that recurses through the `handler-info`s for a transition and calls the associated `resolve` method. Before this, we insert a `requestOneHandlerInfo` method that recurses through `handler-info`s, calling each one's `request` method, which now executes the model hooks, kicking off any network requests.

We add a `requestIndex` property to `transition` that keeps track of how many route's model hooks we have already run (akin to how `resolveIndex` presently keeps track of how many routes we have resolved).

#### Ember.js (emberjs/ember.js#12415)

The default model hook uses `requestIndex` (instead of `resolveIndex`) to grab `modelPromise` (instead of `context`) from the `handler-info`.

`Route#modelFor` returns a promise.

#### Usage

Routes that don't depend on another route's data likely don't need any changes, but their requests are sent in parallel:

```javascript
// app/routes/parent.js
export default Ember.Route.extend({
  model() {
    return Ember.$.get('/api/foo');
  }
});

// app/routes/parent/child.js
export default Ember.Route.extend({
  model(params) {
    return Ember.$.get(`/api/foo/${params.id}`);
  }
});
```

Routes that depend on another route's data must consume that data as a promise:

```javascript
// app/routes/parent.js
export default Ember.Route.extend({
  model(params) {
    return Ember.$.get(`/api/foo/${params.id}`);
  }
});

// app/routes/parent/child.js
export default Ember.Route.extend({
  model() {
    return this.modelFor('parent').then((parentModel) => {
      return Ember.$.get(`/api/bar/${parentModel.get('baz')}`);
    });
  }
});
```

Routes whose `afterModel` hook operates on the resolved model must now consume a promise:

```javascript
export default Ember.Route.extend({
  model(params) {
    return Ember.$.get(`/api/foo/${params.id}`);
  },
  afterModel(modelPromise, transition) {
    return modelPromise.then((data) => {
      if (data.abort) {
        this.transitionTo('fiz');
      }
    }
  }
});
```

# Drawbacks

- This is a major breaking change.
- Having `Route#modelFor` return a promise may be confusing.

# Alternatives

- Tie this paradigm change to something opt-in like routable components where people will already have to change their behavior.
- Maintain it behind a feature flag on the Canary branch which you can toggle on and off via config until we can land it.
- Provide some built-in hooks so that we're able to reach in and change this behavior as an addon in a safer way.
- The worlds most absurd monkeypatch–which would also be packaged as an addon.
- Maintain a patch and build our own custom Ember.

# Unresolved questions


- Should `modelFor` always return a promise? Should we instead add a `modelPromiseFor` method?
  - If `modelFor` returns a promise, how do we make this an easy change for developers?
  - If adding `modelPromiseFor`, what do calls to `modelFor` return before the model's promise has been fulfilled?
