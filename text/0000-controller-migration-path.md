- Start Date: (fill me in with today's date, YYYY-MM-DD)
- Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Migration away from controllers

## Summary

 - Provide alternative APIs that encompass existing behavior currently implemented by controllers.
 - Out of scope:
   - Deprecating and eliminating controllers -- but the goal of this RFC is to provide a path to remove controllers from Ember altogether. 
 - Not Proposing
   - Adding the ability to access more properties than `model` from the route template
   - Adding the ability to invoke actions on the route.
   
This'll explore how controllers are used and how viable alternatives can be for migrating away from controllers so that they may eventually be removed.

## Motivation

 - Many were excited about [Routable Components](https://github.com/ef4/rfcs/blob/routeable-components/active/0000-routeable-components.md).
 - Many agree that controllers are awkward, and should be considered for removal (as mentioned in many #EmberJS2019 Roadmap blogposts).
 - There is, without a doubt, some confusion with respect to controllers:
   - should actions live on controllers?
   - how are they different from components?
   - controllers may feel like a route-blessed component (hence the prior excitement over the Routable-Components RFC)
   - controllers aren't created by default during `ember g route my-route`, so there is a learning curve around the fact that route templates can't access things on the route, and must create another new file, called a controller.


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
