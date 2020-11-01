---
Start Date: 2017-07-20
RFC PR: https://github.com/emberjs/rfcs/pull/237
Ember Issue: (leave this empty)

---

# Summary

This RFC proposes the deprecation of the following classes:

- `Ember.OrderedSet`
- `Ember.Map`
- `Ember.MapWithDefault`

These classes need to be moved to an external addon given they are private classes and unused in Ember.js itself.

# Motivation

These classes have not been used in Ember itself for a while now. They have always been private but they are used in a few addons, and in particular Ember Data is using them.

# Transition Path

`Ember.Map` and `Ember.MapWithDefault` will be deprecated and not extracted, but not before the fix mentioned in the following paragraph is landed in Ember Data. There is already an addon with `Ember.OrderedSet` extracted ([@ember/ordered-set](https://github.com/emberjs/ember-ordered-set)).

Ember Data is quite likely the biggest project using these classes. There is already a PR that needs merging before deprecating `Ember.Map` and `Ember.MapWithDefault` https://github.com/emberjs/data/pull/5255. Ember Data still needs to migrate to `@ember/ordered-set` to its relationship logic.

Once Ember Data is updated to not use the classes from Ember, and that fix is released, the `Ember.Map` and `Ember.MapWithDefault` can be deprecated in Ember itself.

# How We Teach This

These classes being private would make this simple than other deprecations. People were not supposed to be using a private API and the few that were, would just need to use a new addon.

This should not impact many codebases.

# Drawbacks

This requires cooperation with Ember Data, the main user of these classes. It would be nice to have moved Ember Data to using the addon before releasing Ember with the deprecation so the average user does not see any deprecation warning.

# Alternatives

Other option would be moving these classes to Ember Data itself or leaving things as they are now.

# Unresolved questions
