- Start Date: 2017-07-20
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This PR proposes the deprecation of the following classes:

- `Ember.OrderedSet`
- `Ember.Map`
- `Ember.MapWithDefault`

These classes need to be moved to an external addon given they are private classes and unused in Ember.js itself.

# Motivation

Theseclasses has not been used in Ember itself for a while now. They have always been private but they are used in a few addons, and in particular Ember Data is using them.

# Transition Path

The classes will be extracted to an addon and the ones in Ember will be deprecated.

# How We Teach This

These classes being private would make this simple than other deprecations. People were not supposed to be using a private API and the few that were, would just need to use a new addon.

This should not impact many codebases.

# Drawbacks

This requires cooperation with Ember Data, the main user of these classes. It would be nice to have moved Ember Data to using the addon before releasing Ember with the deprecation so the average user does not see any deprecation warning.

# Alternatives

Other option would be moving these classes to Ember Data itself or leaving things as they are now.

# Unresolved questions
