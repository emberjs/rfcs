- Start Date: 2017-12-07
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

I'd like to propose this change: The model hook will always run, even for dynamic segments that already have their models loaded.

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

The fix I would like to propose is to always run the model hook when entering a route. The params would resolve to the model's ID. To be more specific, the params arguments would be generated from the route's serialize function if passed a model or object.

The model hook can then make a decision if it needs to refetch the model. This seems to be the approach Ember-data has taken with `findRecord`, it's aware if the model has already been loaded, and if it is it will do a background reload. I like this thinking because the it leaves the data fetching decision up to the model layer.

Also, always running the model hook simplifies some complexity since dynamic and non-dynamic routes will now have the same behavior.

# How We Teach This

The guides would need to be updated. I think a blog post would help explain the change to behavior.

# Drawbacks

This is a breaking change. It could introduce un-intended side effects if model hooks suddenly start running when they previously were not.

If applications are currently linking to string IDs we can't pass that through the route's `serialize` function.  Would those get turned into params based off the order of the dynamic segments in the route's path?

# Alternatives

I'd love to make an addon that can proof-of-concepts this. I would need some help though.

Another alternative is that this could be something that's done when calling `link-to` or `transitionTo`, similar to how folks are linking to IDs today.

```hbs
{{! run-model-hook takes a model and returns an id, which causes
    the model hook to run when the link is clicked }}
{{link-to "post.show" (run-model-hook post)}}
```

```js
// options for the transition.
this.transitionTo('post.show', model, { reload: true });
```

My gut says these are bad because they putting model loading knowledge outside of the route.

# Unresolved questions
