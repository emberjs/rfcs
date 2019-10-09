- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# URL Primitives

Supercedes [Add queryParams to the router service](https://github.com/emberjs/rfcs/pull/380)

Notes from rwjblue:

This RFC should include a bit more detail:

a comprehensive list of the ways the current QP system can be used (the use cases at least), and how you would do the same things in the new APIs
examples of how we would teach the new API (e.g. for the guides)
examples of how current QP related addons would plausibly function over the new APIs

I'd also like to see an exploration of why the core has to do this at all, what would a minimal API look like to allow this entire RFC to be implementable in user space? From my perspective, there are a few fundamental things that might unblock that:

adding a public API to access Router.map defined metadata
adding a hook that can be used to customize the target of all .transitionTo / .replaceWith invocations (intercepting the QP properties, and setting state on the QP service)
adding a hook that can be used to customize how a URL is translated back into router params (intercepting the URL string and setting state on the QP service)
I'm willing to accept that this might be naive, but it seems like an interesting (smaller, more customizable, and easier to implement) alternative...

## Summary

> One paragraph explanation of the feature.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
outcome?

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
