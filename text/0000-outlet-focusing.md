- Start Date: 2015-06-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Add a `focusNode` property to `Ember.Route` which will enable programmatically setting the focus to the specified element after a navigation event and a completed transition. By default this will be `undefined` which will result in the first element associated with the route being focused. There will be a new private API `focus(morph)` function which receives the associated morph node, reads this property, and sets the focus appropriately. The default implementation should make all Ember applications more accessible by default.

For bonus points, add an event that is fired at a `route` on "focus" which allows an additional hook for behavior correlated to focusing a route, such as addressing scroll position. (This is distinct from `enter` and `activate`.)

# Motivation

When a screen reader user navigates through an Ember site using `{{link-to}}` helpers we do nothing to make it clear that a portion of the screen has changed. We override the default behavior of the link and give no explanation of what occurred: just silence.

This is a tremendously bad user experience, and something that we need to fix. Ember should have the goal of being accessible by default and this RFC is the first of many to hopefully make it so.

# Detailed design

- [Gist with a mock implementation.](https://gist.github.com/nathanhammond/f48a793042d5100fbe89)
- [Example working application.](https://github.com/nathanhammond/ember-outlet-a11y)

## Technical details

- This attempts to grab the "primary" outlet connection for that particular route. Custom `renderTemplate()` functions with only named outlets can make this not play nicely. It's not configurable in my mock implementation, but it could be assertable that there always be a "main" outlet.
- The best experience requires that the first node in every template be a grouping element containing the entire template. I propose that we deprecate non-`div`-wrapped `{{outlet}}`s with a global opt-in to `div`-wrapped outlets which removes that deprecation. This makes the behavior more accessible by default, enforces the concept of "nested routing" and makes it easier for applications to adopt liquid-fire (and guaranteeing an element handle for `.liquid-child`). *I expect this to be the most controversial piece of this RFC and believe the tradeoff to be worth it.*
- Currently it relies on the `Ember.Route`'s `enter` hook to tie the route to the transition. This misses no-op transitions. The mock implementation is the weak link here, not the concept.
- It gets clever when you're navigating back to the index route of a route you're presently in and sets the focus to the "outer" route.

## Accessibility thoughts

Programmatically setting focus is often done poorly and as a result it is generally recommended against in the assistive tech community. This behavior is far from being any kind of standard and we will be charting new waters. This is something we should take on, but we will likely get pushback on any decision that we make, either way. We should make sure we involve people outside of just the Ember community.

# Drawbacks

- Changes the focus behavior for all users, not just screen reader users.
- Setting focus has style and scrolling consequences. [This is addressable.](http://jsbin.com/faxazedaci/edit?js,output)
- Dictates outlet markup (wrapping in `div`), reducing their flexibility.
- Requires more rigidity on named `{{outlet}}`s to require at least one "main" outlet.

# Alternatives

We could simply set focus to `document.body` at the end of every transition. We get some of the same behavior, little of the complexity, none of the possible design wins, and both of the worst drawbacks (the first two).

It's also possible that this can be implemented by exposing lower-level generic hooks inside of Ember, Extensible Web Manifesto style. This would allow for the functionality to exist inside of an addon instead of being a core part of Ember. This doesn't accomplish one of my primary goals, however: accessible by default.

I don't consider not coming up with a solution an actual possibility.

# Unresolved questions

The design space has been pretty well explored. There are a few more implementation details to make sure that things play nicely with all of the edge cases and bikeshedding on API names, but those can be addressed after community feedback on the feature's existence.
