- Start Date: (2017-06-12)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Right now there's no public API surrounding the LinkComponent's `_invoke` method. Adding this would allow for addons and applications to extend the LinkComponent and easily add logic gates around the `_invoke` as well as allow for added functionality prior to the transition.

# Motivation

- Analytics and click tracking
- Open routes in modals based on environment

# Detailed design

Right now the `init` directly hooks the user-defined event type to fire the `_invoke` which is private.  `_invoke` then directly works on the transition to the new route.

A new public method `eventAction` will be bound during `init` instead of the `_invoke`.  `eventAction` simply calls `_invoke`.

Extended components can now override `eventAction`, perform additional logic such as metrics/click-tracking and also logically gate a call to `_super` which would continue execution to transition the route as normal.

# How We Teach This

Recommend making this change as unobtrusive as possible and silently adding the new method to the documentation.  Several very common addons will likely benefit from this change and can likely move off of some private overrides currently being used.

Most Ember consumers will benefit from this change via addon updates, rather than direct usage.

# Drawbacks

Adding additional features is a difficult sell as LinkComponent is likely set to be reworked into a more modular approach.

# Alternatives

Also considered adding a `shouldInvoke` method that returns a boolean and then having `_invoke` call `shouldInvoke` at the head of the function.  `_invoke` would then NOOP if `shouldInvoke` returns `false`.

This approach is less clear as a means to add additional pre-invoke logic.

While Ember does use `should` prefixes to a certain extent, it is far from a common naming scheme currently.

# Unresolved questions

Is it worth making this change, albeit a very small one, in light of current mentalities to break LinkComponent down into a collection of more specific micro-components?
