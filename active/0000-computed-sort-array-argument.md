- Start Date: 2015-08-26
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

In Ember 2.x, `Ember.computed.sort` is the one and only computed macro for sorting a list. Its current API is built assuming that the sort order may change at runtime, leaving no way to easily create a declarative sort macro that you know is immutable. This RFC proposes adding an additional argument type for the `sortDefinition` parameter to make it easier to declare and to ensure sort order is immutable.

# Motivation

I'm hoping to satisfy two primary annoyances with this RFC. First, `Ember.computed.sort` can be used as a singular definition as the sort order will be the value of the second argument, you will not have to look for another definition on the class, thus improving code readability. In our app, we often have the following:

```javascript
var NewestPostsController = Ember.Controller.extend({
  posts: null,
  newestPosts: Ember.computed.sort('posts', '_newestPostsOrder'),
  _newestPostsOrder: ['createdOn:desc']
});
```

The second annoyance is I know that if `_newestPostsOrder` was inadvertently changed, it could cause `newestPosts` to be incorrect. I realize this would just be a bug in our app, but having the ability to define an immutable sort order would give the same peace of mind as a readOnly computed property.

# Detailed design

Right now the second argument for `Ember.computed.sort`, `sortDefinition`, accepts two values, a string and a function. I'm proposing we add array as a third allowed argument type. The array would expect the exact same sort properties format as it does when using a string as a dependent key. Reusing the above example,

Before:
```javascript
var NewestPostsController = Ember.Controller.extend({
  posts: null,
  newestPosts: Ember.computed.sort('posts', '_newestPostsOrder'),
  _newestPostsOrder: ['createdOn:desc']
});
```

After:
```javascript
var NewestPostsController = Ember.Controller.extend({
  posts: null,
  newestPosts: Ember.computed.sort('posts', ['createdOn:desc'])
});
```

This design would not break semver as it's an additional argument type that was previously not supported.

# Drawbacks

I can think of two drawbacks. First and foremost, the array argument type will become reserved for this functionality. It could not easily be reused for a new extension in the future. Second, it's ultimately just a more convenient way of doing something that is already functionally possible.

# Alternatives

No real alternatives were considered. The only thing that came to mind as an alternative was adding an additional computed macro for this use case (EG `Ember.computed.readOnlySort`).

# Unresolved questions

I don't believe there are any unresolved questions at this time.
