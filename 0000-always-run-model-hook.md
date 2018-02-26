- Start Date: 2017-12-07
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The model hook will always run, even for dynamic segments that already have their models loaded.

# Motivation

Currently the router will not run the model hook when transition to a route with a dynamic segment that already has its model loaded. This behavior is well intended, if we are using `{{link-to "post.show" post}}` then there's no reason to run the `post.show` model hook, the model has already been loaded!

The guides do a great job [explaining](https://guides.emberjs.com/v2.17.0/routing/specifying-a-routes-model/#toc_dynamic-models) this feature.

Before I started using JSON:API I thought this feature made a lot of sense, it would only fetch models when the application didn't have them. However, over the last year there's been a pattern I've seen in a couple Ember applications that this behavior complicates.

Let's say we have a `posts.index` route that lists all of the posts. It's model hook:

```js
// app/routes/posts/index.js

export default Ember.Route.extend({
  model() {
    return this.store.findAll('post');
  }
});
```

And then we also have a `posts.show` route that shows a specific post. When viewing a post we want to side load some additional data, like the post's comments.

```js
// app/routes/posts/show.js

export default Ember.Route.extend({
  model(params) {
    return this.store.findRecord('post', params.post_id, {
      include: ['comments.author'],
      reload: true
    });
  }
});
```

This creates an issue: Since the post model is already loaded, when we click a link on the index page to view a post, the model hook never runs. This causes the post to either render without comments, or worse, render with an async call to comments and then N+1 async calls to those comment's authors.

To code around this behavior a popular pattern I've seen is to link to the post ID. Since the ID is not the model, Ember will run the model hook and the correct data will be loaded.  

```hbs
{{link-to "posts.show" post.id}}
```

However, this isn't intuitive. From looking at this `link-to` it's not clear why ID is being used.

As an aside, I posted [this poll](https://twitter.com/ryantotweets/status/938446826117746689) on Twitter and from the answers it appears that other folks find this confusing or use the ID work around. The Ember crowd on Twitter represents folks who are in-tune with the framework, if this many experts feel this way I think it's a sign we can improve ergonomics.

# Detailed design

We would introduce a new route base class which would default to always running the model hook. In order to avoid confusion, this new route class would have a new hook called `findModel` that would always run.

In addition to always running the `findModel` hook when transitioning, this new route class would:

* Return a POJO, allowing you to pass arbitrary names to the template. Template rendering will wait for all promises in the POJO to fulfill.
* Parent/child routes would not waterfall. Child routes would fire their `findModel` hooks before parent routes have resolved. This would cause `modelFor` to return a promise.

The new route base class would allow us to implement these new ideas without worrying about the API of the existing route class. Developers can swap out their route classes when they are ready to adapt this new behavior.

# Drawbacks

If applications are currently linking to string IDs how do we pass them through the route's `serialize` function.

# Alternatives

### Always run model hook

Always run the model hook on the current route class when entering a route. The params arguments would be generated from the route's serialize function if passed a model or object.

The model hook can then make a decision if it needs to refetch the model. This seems to be the approach Ember-data has taken with `findRecord`, it's aware if the model has already been loaded, and if it is it will do a background reload. This leaves the data fetching decision up to the model layer.

# Unresolved questions

I'd love to make an addon that can proof-of-concepts this. I would need some help though.
