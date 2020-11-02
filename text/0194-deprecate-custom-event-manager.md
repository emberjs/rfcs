---
Start Date: 2016-12-24
RFC PR: https://github.com/emberjs/rfcs/pull/194
Ember Issue: https://github.com/emberjs/ember.js/issues/14754

---

# Summary

Support for component `eventManger`s is a seldom used feature and should
be deprecated.

# Motivation

We should strive to simplify the Ember API and source code where possible. As
the custom `eventManager` feature is rarely used in apps, we should deprecate
it.

# Detailed design

We'll introduce a deprecation warning which will be displayed when a component
defines an `eventManager` property or when `canDispatchToEventManager` is set to
true on `EventDispatcher`. The warning will have a target version of `3.0`.

If required, we can create an addon which extends the `EventDispatcher` allowing
for opt-in custom `eventManager`s in Ember apps.

# How We Teach This

As this is a seldom used feature, we can simply note the deprecation in a
future release blog post.

# Drawbacks

This adds a little more churn for apps that rely on this feature.

# Alternatives

This feature was [recently made pay-as-you-go](https://github.com/emberjs/ember.js/pull/14756),
so the immediate performance concerns have been addressed. We could decide to
leave this in the framework as an opt-in feature.
