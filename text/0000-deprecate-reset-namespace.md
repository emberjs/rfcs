- Start Date: 2017/04/18
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC suggest the deprecation and eventual removal of the `resetNamespace` option
in routes.

# Motivation

This little known feature is mostly used to have shorter route names on nested routes.
It can be used to move a route hierarchy to a more nested location without having to rename
all places where that route was referenced by its name (mostly `{{link-to}}`, but not exclusively).

However reseting the namespace of a route prevents developers to intuitively understand
the placement of a route in the nesting hierarchy of the app by simply reading its name,
making the mental model and the task of finding a a route in the file system more complex.
This issue is going to be amplificated by the deeply nested file structure of the
upcoming Module Unification.

This feature has been deemed dangerous already and the learning team has tried to reduce
it's usage by [intentionally removing it from the guides](https://github.com/emberjs/guides/pull/1006),
much like it happened with the already deprecated `route.resource` method.

# Detailed design

Follow the usual process for other deprecations. Add the deprecation and targetEmber 3.0
for its complete removal. That deprecation will take developers to an entry in the
deprecations guide explaining how to stop using it.

# How We Teach This

Since this feature has already been removed from the guides, an entry in the deprecations
guide explaining how to stop using it, which is very straightforward, is the only
action point.

# Drawbacks

There is an unknown number of apps that use this feature and its deprecation will cause
some churn. The older the project is, the highest the chance of the app using this feature.

# Alternatives

Keep the feature and commit to maintain it in the 3.0 cycle.

