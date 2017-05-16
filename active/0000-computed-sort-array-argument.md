- Start Date: 2015-08-26
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

In Ember 2.x, `Ember.computed.sort` is the one and only computed macro for sorting a list. Its current API is built assuming that the sort order may change at runtime, leaving no way to easily create a declarative sort macro that you know is immutable. This RFC proposes adding an additional computed macro for declarative and immutable sorting.

# Motivation

I'm hoping to satisfy two primary annoyances with this RFC. First, providing a way to declare a computed sort macro as a singular definition (one line of code). As opposed to `Ember.computed.sort` which requires two declarations to use. It's my belief this will improve readability for some use cases. In our app, we often have the following:

```javascript
var NewestPostsController = Ember.Controller.extend({
  posts: null,
  newestPosts: Ember.computed.sort('posts', '_newestPostsOrder'),
  _newestPostsOrder: ['createdOn:desc', 'upvoteCount']
});
```

The second annoyance is I know that if `_newestPostsOrder` was inadvertently changed, it could cause `newestPosts` to be incorrect. I realize this would just be a bug in our app, but having the ability to define an immutable sort order would give the same peace of mind as a readOnly computed property.

# Detailed design

With this RFC I'm proposing we introduce an additional computed macro, `Ember.computed.sortBy`, to fill the gap. The argument style would be similar to that of `Ember.computed.sort`. However, with `Ember.computed.sortBy` all non-primary sort orders (secondary, tertiary, etc) would be defined as additional arguments. `Ember.Enumerable.sortBy` serves as existing precedence for this pattern. Reusing the above example,

Before:
```javascript
var NewestPostsController = Ember.Controller.extend({
  posts: null,
  newestPosts: Ember.computed.sort('posts', '_newestPostsOrder'),
  _newestPostsOrder: ['createdOn:desc', 'upvoteCount']
});
```

After:
```javascript
var NewestPostsController = Ember.Controller.extend({
  posts: null,
  newestPosts: Ember.computed.sortBy('posts', 'createdOn:desc', 'upvoteCount')
});
```


# Drawbacks

The only draw back I could come up with is that this increases the surface of the API and is ultimately just a more convenient way of doing something that is already functionally possible.

# Alternatives

When I originally drafted this RFC, `Ember.computed.sortBy` was the alternative, but it's became the clear favorite. So flipping this on it's head, the alternative here is modify the existing `Ember.computed.sort` to have this behavior. Additionally, this could be implemented as an addon or perhaps added to [ember-cpm](https://github.com/cibernox/ember-cpm).

# Unresolved questions

I don't believe there are any unresolved questions at this time.
