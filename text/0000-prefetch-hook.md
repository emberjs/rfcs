- Start Date: 2015-10-01
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The waterfall way in which Ember resolves a transition's model hooks can result in unnecessary blocking.
Performance wins can be achieved by allowing routes to decouple initiating network requests from waiting for their completion.

# Motivation

Currently, the `model` hook is Ember's way for routes to request data.
However, a route's `model` hook is not entered until it's parent route has resolved.
This is suboptimal for nested application structures.
Child routes are forced to wait for their parents to receive data before they may make requests for their own data.
This occurs even when the child doesn't make use of its parents' data.

This RFC proposes adding a new `Route#prefetch` hook, which is used to allow routes to make requests in parallel.
Child routes that make use of `prefetch` will resolve faster since their data can resolve at the same time as (or before) their parents'.
The semantics and ordering of `Route`'s existing model hooks (`beforeModel`, `model`, `afterModel`) are preserved.

See this demo for a visualization: http://nickiaconis.github.io/ember-parallel-model-demo/

# Detailed design

A `prefetch` hook is added to `Route`.
It takes the same parameters as the `model` hook.
Like the `model` hook, it is not called if an object is passed to the transition.

```javascript
App.PostRoute = Ember.Route.extend({
  prefetch(params) {
    return Ember.$.get(`/api/posts/${params.id}`);
  }
});

App.PostCommentsRoute = Ember.Route.extend({
  prefetch(params, transition) {
    return Ember.$.get(`/api/posts/${transition.params.post.id}/comments`);
  }
});
```

The default functionality of the `model` hook is modified to return the prefetched data if it exists.
As such, a route that defines a `prefetch` hook is not required to define a `model` hook.

A `prefetched` property is added to `Route`.
It is used to pass a route's prefetched data from the `prefetch` hook to the default `model` hook.
`prefetched` will always be a promise.

`prefetched` can also be used to get a route's own data.
This is useful for expanding upon the prefetched data in the route's `model` hook.

```javascript
App.PostCommentsRoute = Ember.Route.extend({
  prefetch(params, transition) {
    return Ember.$.get(`/api/posts/${transition.params.post.id}/comments`);
  },
  
  async model() {
    return {
      OP: this.modelFor('post')).author,
      comments: await this.prefetched
    };
  }
});
```

# Drawbacks

- Ember's API becomes larger.

# Alternatives

- Provide something like `Router#willTransition`, but that is triggered on redirects, so this can be built as an addon. It could be triggered on either all transitions (`Router#willChangeURL`?) or only redirects (`Router#willRedirect`? to supplement `willTransition`). Otherwise, building this as an addon must change the functionality of `Router#willTransition` in order to function properly.

# Unresolved questions

- Is there a use case for the API to include a method like `modelFor`, but for `prefetch`? (`prefetchedFor`/`asyncModelFor`)
