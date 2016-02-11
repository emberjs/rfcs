- Start Date: 2016-02-11
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes introducing a new construct called a `RouteSerializer` to replace the existing `Route#serialize` method.

# Motivation

As we move towards an increasingly asynchronous world, we need to separate knowledge about how to _**link** to a route_ and how to _**enter** a route_. Linking to a route should be able to happen _before_ a route object is instantiated, but entering a route can only occur after a route object is instantiated. However, in our current reality, a route object must be instantiated before being able to link to _or_ enter a route.

By separating these concerns, we can preemptively load the information on how to link to a route without also requiring all the knowledge of how to enter that route. This would be beneficial in both the asynchronous and synchronous worlds by allowing us to defer work.

Since the `serialize` method is the only method currently used by the `Route` class to define how to link to a route, the proposal is to extract this into a standalone construct.

_**Note:** there are also changes that need to be made in router.js for the preemptive loading proposed here to actually work, but we can prepare for that future world by creating a separation of concerns now._

# Detailed Design

Since the current API is a simple function, the new API will also be a simple function that mirrors the signature of the original. However, in order to make sure future compatibility and API changes can be done smoothly, it will require being wrapped in a helper function, `Ember#routeSerializer`. Here's an example:

```js
// route-serializers/foo.js
import Ember from 'ember';
const { routeSerializer } = Ember;

export default routeSerializer(function(model, params) {
   // do serialization stuff
});
```

This means that refactoring existing code should be super simple. Here's the example currently given in the Ember docs (updated to reflect Ember-CLI):

```js
// app/routes/post.js
import Ember from 'ember';
export default Ember.Route.extend({
  model: function(params) {
    // the server returns `{ id: 12 }`
    return Ember.$.getJSON('/posts/' + params.post_id);
  },

  serialize: function(model) {
    // this will make the URL `/posts/12`
    return { post_id: model.id };
  }
});
```

Here is that same code refactored for the proposal:

```js
// app/routes/post.js
import Ember from 'ember';
export default Ember.Route.extend({
  model: function(params) {
    // the server returns `{ id: 12 }`
    return Ember.$.getJSON('/posts/' + params.post_id);
  }
});

// app/route-serializers/post.js
import Ember from 'ember';
export default Ember.routeSerializer(function(model) {
  // this will make the URL `/posts/12`
  return { post_id: model.id };
});
```

# Migration Plan

Even though the refactoring needed here is easy, we still need a clear (though simple) migration plan.

The first step will be to introduce the `RouteSerializer` concept into both Ember and Ember-CLI. Once that is done, we can deprecate `Route#serialize` over the remainder of the 2.x series to give developers the time to update their code base. We can then remove support in 3.x.

As noted in the "Motivation" section, there is still work to be done in router.js in order to support this separation of concerns. Due to this, the initial implementation of `RouteSerializer` will essentially be a polyfill that proxies the corresponding `Route#serialize` property. This will set us up for an internal migration at a later point to actually separate the two; this, however, should not affect developers as it will be internal.

# Pedagogy (How We Teach This)

Once the `RouteSerializer` concept is introduced, the Ember guides will need to be updated to reflect this new construct. Those changes should be relatively straightforward as shown in the example above. This will help introduce the feature to new users and those users that haven't used `Route#serialize` before.

For existing users, we can introduce this feature through deprecation warnings (as mentioned above). The deprecations should briefly introduce `RouteSerializers` and point to an appropriate deprecation guide that explains how to migrate.

# Drawbacks

- Adds another function to the API.
- Adds another type of construct to manage.

# Alternatives

- Introduce a more holistic construct. Instead of keeping the API as a function, we could introduce a class that would represent all the information needed to link to a route. Since there is not currently any other information needed, this seems overkill.
- Don't do this and continue loading and instantiating all route information upfront. This is bad for performance and keeps concerns coupled.

# Unresolved Questions

- Do we wish to apply a similar approach for default query params? And if so, do we wish to incorporate that approach into this new construct?
- Where should the new `routeSerializer` method live in a modules world? `ember-route-serializer`?
