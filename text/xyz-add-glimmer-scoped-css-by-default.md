---
stage: accepted
start-date: # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

# Make Scoped CSS the default in 'template-tags'

## Summary

A component is defined as a unit of UI with its action and style. 

```gjs
<template>
  <h1>Scoped CSS</h1>
    <p>How is scoped done in Emberjs template-tag?</p>

  <style>
      p { font-family: Roboto};
    </style>
</template>
```
Based on my observation while using `template-tags` to author
components, the style tag inside `template-tags` is global, which I think
should not be because it may affect other components. Also, libraries or
imported components might have used the same style name, so the issue will become nested
and, therefore won't be easy to trace. Even if one works around this using a
unique class wrapper for each component, the possibility of collision is still
present, and uniqueness is not guaranteed project-wide.


## Motivation

To make `template-tags` fulfil all three things (UI, action, style) that
make an excellent component authoring tool. This will make the authoring of
components with `template-tags` much better and safer.

Making scoped-CSS the default will improve the developer experience by not having
to fight spill CSS that is affecting other components.


## Detailed design

There is an implementation already.  [glimmer scoped-css](https://github.com/cardstack/glimmer-scoped-css)

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here. 

> Please keep in mind any implications within the Ember ecosystem, such as:
> - Lint rules (ember-template-lint, eslint-plugin-ember) that should be added, modified or removed
> - Features that are replaced or made obsolete by this feature and should eventually be deprecated
> - Ember Inspector and debuggability
> - Server-side Rendering
> - Ember Engines
> - The Addon Ecosystem
> - IDE Support
> - Blueprints that should be added or modified

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

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
