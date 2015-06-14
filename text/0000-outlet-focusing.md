- Start Date: 2015-06-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Add a `focus(morph)` hook to `Ember.Route` which will enable programmatic setting of focus after a navigation event and a completed transition. This should ship with a default implementation that makes all Ember applications more accessible by default. It can then be overridden by a developer to provide behaviors specific to their application's use case. 

# Motivation

When a screen reader user navigates through an Ember site using `{{link-to}}` helpers we do nothing to make it clear that a portion of the screen has changed. We override the default behavior of the link and give no explanation of what occurred: just silence.

This is a tremendously bad user experience, and something that we need to fix. Ember should have the goal of being accessible by default and this RFC is the first of many to hopefully make it so.

# Detailed design

- [Gist with a mock implementation.](https://gist.github.com/nathanhammond/f48a793042d5100fbe89)
- [Example working application.](https://github.com/nathanhammond/ember-outlet-a11y)

## Technical details

- This attempts to grab the "primary" outlet connection for that particular route. Custom `renderTemplate()` functions with only named outlets can make this not play nicely. It's not configurable in my mock implementation, but it could be assertable that there always be a "main" outlet.
- It requires that the first node in every template be a grouping element containing the entire template. We could address this by forcing all `{{outlet}}`s to be wrapped in `div`s by default, but that is a breaking change. Maybe find a way to opt in to that behavior. (We know that this is generally safe because of liquid-fire, but it was also decided against making a change to the top level view during the run-up to 2.0.)
- Currently it relies on the `Ember.Route`'s `enter` hook to tie the route to the transition. This misses no-op transitions.
- It gets clever when you're navigating back to the index route of a route you're presently in and sets the focus to the "outer" route.

## Accessibility thoughts

Programmatically setting focus is often done poorly and as a result it is generally recommended against in the assistive tech community. This behavior is far from being any kind of standard and we will be charting new waters. This is something we should take on, but we will likely get pushback on any decision that we make, either way. We should make sure we involve people outside of just the Ember community.

# Drawbacks

- Changes the focus behavior for all users, not just screen reader users.
- Setting focus has style and scrolling consequences.
- Dictates outlet markup (wrapping in `div`), reducing their flexibility.
- Requires more rigidity on named `{{outlet}}`s to require at least one "main" outlet.
- Makes available the relevant DOM inside of a Route, which could be abused by enterprising developers.

# Alternatives

We could simply set focus to `document.body` at the end of every transition. We get some of the same behavior, little of the complexity, none of the possible design wins, and both of the worst drawbacks (the first two).

It's also possible that this can be implemented by exposing lower-level generic hooks inside of Ember, Extensible Web Manifesto style. This would allow for the functionality to exist inside of an addon instead of being a core part of Ember. This doesn't accomplish one of my primary goals, however: accessible by default.

I don't consider not coming up with a solution an actual possibility.

# Unresolved questions

The design space has been pretty well explored. There are a few more implementation details to make sure that things play nicely with all of the edge cases, but those can be addressed after community feedback on the feature's existence.
