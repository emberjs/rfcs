---
Start Date: 2016-02-11
RFC PR: emberjs/rfcs#120
Ember Issue: https://github.com/emberjs/ember.js/pull/13016

---

# Summary

This RFC proposes replacing the existing [`Route#serialize`](http://emberjs.com/api/classes/Ember.Route.html#method_serialize) method with an equivalent method on the options hash passed into [`this.route` within `Router#map`](http://emberjs.com/api/classes/Ember.Router.html#method_map). The primary goal here is to enable asynchronous Engines by decoupling information about how to link to a route from the actual route object.

# Motivation

As we move towards an increasingly asynchronous world with Engines, we need to separate knowledge about how to _**link** to a route_ and how to _**enter** a route_. Linking to a route should be able to happen _before_ a route object is instantiated, which is the behavior needed to asynchronously load Engines. However, in our current reality, these concerns are coupled and a route object must be instantiated before being able to link to _or_ enter a route.

By separating these concerns, we can preemptively load the information on how to link to a route without also requiring all the knowledge of how to enter that route. This would be beneficial in both the asynchronous and synchronous worlds by allowing us to defer work.

Since the `serialize` method is the only method currently used by the `Route` class to define how to link to a route, the proposal is to extract this method into the space which currently contains the other linking information (e.g., the Router's map).

_**Note:** this separation of concerns will also need to be implemented in router.js for the preemptive loading proposed here to actually work, but we can prepare for that future world by creating a separation of concerns within application code now._

# Detailed Design

Since the current API is a simple function, the new hash option will also be a simple function that mirrors the signature of the original. Here's an example:

```js
// app/router.js
function serializePostRoute(model, params) {
  // serialize the model into the dynamic paths
}

export default Router.map(function() {
  this.route('post', { path: '/post/:id', serialize: serializePostRoute });
});
```

Preserving the current function signature means that refactoring existing code should be simple in most cases. Here's the example currently given in the Ember docs (updated to reflect Ember-CLI):

```js
// app/routes/post.js
import Ember from 'ember';
export default Ember.Route.extend({
  model(params) {
    // the server returns `{ id: 12 }`
    return Ember.$.getJSON('/posts/' + params.post_id);
  },

  serialize(model) {
    // this will make the URL `/posts/12`
    return { post_id: model.id };
  }
});

// app/router.js
export default Router.map(function() {
  this.route('post', {
    path: '/post/:id'
  });
});
```

Here is that same code refactored for the proposal:

```js
// app/routes/post.js
import Ember from 'ember';
export default Ember.Route.extend({
  model(params) {
    // the server returns `{ id: 12 }`
    return Ember.$.getJSON('/posts/' + params.post_id);
  }
});

// app/router.js
function serializePostRoute(model) {
  // this will make the URL `/posts/12`
  return { post_id: model.id };
}

export default Router.map(function() {
  this.route('post', { path: '/post/:id', serialize: serializePostRoute });
});
```

# Migration Plan

Even though the refactoring needed here is easy, we still need a clear (though simple) migration plan.

The first step will be to introduce the new option into the Router's callback `route` function. Once that is done, we can deprecate `Route#serialize` over the remainder of the 2.x series to give developers the time to update their code base. We can then remove support in 3.x.

As noted in the "Motivation" section, there is still work to be done in router.js in order to support this separation of concerns. Due to this, the initial implementation of this new option will essentially be a polyfill that proxies to the corresponding `Route#serialize` property internally. This will set us up for an internal migration at a later point to actually separate the two; this, however, should not affect developers as it will be internal.

# Pedagogy (How We Teach This)

Once the new option is introduced, the Ember guides will need to be updated to reflect this. Those changes should be relatively straightforward as shown in the example above. This will help introduce the feature to new users and those users that haven't used `Route#serialize` before. Since inline serializers in the router map can be distracting to understanding the general layout of a codebase, we should teach them as defined outside the map itself (as in the code example in this RFC).

For existing users, we can introduce this feature through deprecation warnings (as mentioned above). The deprecations should briefly introduce the new option and point to an appropriate deprecation guide that explains how to migrate.

# Drawbacks

- Adds another option to the Router map. Though this is largely mitigated due to the fact that this feature is not in wide use currently.
- Can be sort of ugly to format.

# Alternatives

- Introduce a standalone module to represent the `Route#serialize`. This was the first proposal of this RFC and there is much opposition to introducing yet another construct for Ember-CLI and developers to manage. The approach proposed above avoids this major drawback.
- Introduce a holistic construct to represent route linking information. Instead of introducing a new option as a function, we could introduce a class that would represent all the information needed to link to a route. Since there is not currently much other information needed, this seems overkill and would suffer similar opposition as the first alternative.
- Don't do this and continue loading and instantiating all route information upfront. This prevents us from improving performance by keeping concerns coupled with prevents introducing async engines. It also requires all Route classes be instantiaed upfront.

# Unresolved Questions

- Do we wish to apply a similar approach for default query params? And if so, do we wish to incorporate that approach into this new construct?
