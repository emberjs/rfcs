- Start Date: 2017-07-25
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Align Ember Data module API with [RFC #176](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md).

# Motivation

This document proposes changes to the modules exported by Ember Data to make it consistent with the changes to the Ember module API proposed in [RFC #176](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md).

# Detailed design

The 

Ember Data will stick with 1 top level namespace of `ember-data`.
Under it there are 6 nested module  namespaces where exposed components could live.
Someday these namespaces could be moved into their own top level scoped package (for example `ember-data/serializer` could become `@ember-data/serializer`) however,
that is not being proposed at this time. 

# How We Teach This

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

How should this feature be introduced and taught to existing Ember
users?

# Drawbacks

Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.

# Alternatives

What other designs have been considered? What is the impact of not doing this?

This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
