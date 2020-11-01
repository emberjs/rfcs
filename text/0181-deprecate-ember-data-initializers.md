---
Start Date: 2016-11-22
RFC PR: (leave this empty)
Ember Issue: (leave this empty)

---

# Summary

The goal of this RFC is to remove the `data-adapter`, `injectStore`,
`transforms`, and `store` Ember application initializers that Ember Data injects
into apps.  The `ember-data` initializer will not be changed and any code
that previously depended on the ordering of these initializers (via
the `before` or `after` properties on an initalizer) can be
changed to use the `ember-data` initializers for ordering.

# Motivation

The initializers `data-adapter`, `injectStore`, `transforms`, and
`store` have not been used by Ember Data since
[Apr 8, 2014](https://github.com/emberjs/data/commit/d25e23f622a3677b8372db535b2ab824ad306a16). However,
they are still injected into every Ember app that depends on Ember
Data because existing apps may depend on these initializers
for ordering their own initializers to run before or after Ember
Data's setup code.

Removing these initializers will help reduce the amount of code Ember
Data needs to support.

Since these initializers are noop functions that run after the
`ember-data` initializer, any initializers that depends on one of the
deprecated initializers listed in this rfc can easly be replaced by
depending on the `ember-data` initializer instead.

# Detailed design

Ember Data's instance initializer will start checking for any
initializers whose `before` or `after` properties depend on one of
these deprecated initalizer. If it finds an initalizer that references
one of the deprecated initalizers, Ember Data will then log a
deprecation message that states the name of the offending initalizers
and suggest changing the `before` or `after` property (the deprecation
message will refer to the correct property dynamically) to depend on
Ember Data instead.

This deprecation message will continue to appear until Ember Data
3.0.0 when these initalizers and the deprecation code will be finally
removed.


# How We Teach This

This change should have no impact on how we teach Ember or Ember
Data. The initalizers that will be removed have been unused for a long
time and are not mentioned anywhere in today's guides or API docs.

Users who need to run initalizer code before or after Ember Data
injects the store into routes should be taught to use `before:
'ember-data'`, or `after: 'ember-data'` on their initializers.

# Drawbacks

- This change will require users who depend on these deprecated initalizers to update their code.

# Alternatives

- We could leave the noop initalizers in Ember Data and continue to support them in Ember Data 3.0.0 and beyond.

# Unresolved questions

None
