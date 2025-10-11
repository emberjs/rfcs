---
stage: accepted
start-date: 2025-10-11T16:34:13.823Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
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

<!-- Replace "RFC title" with the title of your RFC -->

# Deprecate broccoli CSS pipeline

## Summary

With vite becoming default we should use it's CSS pipeline as the default experience. We can advise in a deprecation guide that `/@embroider/virtual/app.css` exists if they still need to use it but new apps should be using the vite CSS pipeline by default.

## Motivation

> Why are we doing this? What use cases does it support? What is the expected
No longer need to maintain ember specific CSS tooling like ember-cli-sass ember-css-modules etc. Vite comes with sane defaults to use postcss, sass, less, stylus etc out of the box.
We also wont have to keep explaining to people how to opt in to vite's CSs pipeline.

## Detailed design

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

Replace the CSS import in our app blueprint to the file on disk.

```html
<!-- replace this -->
<link integrity="" rel="stylesheet" href="/@embroider/virtual/app.css">

<!-- with this -->
<link integrity="" rel="stylesheet" href="/app/styles/app.css">
```

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

As part of a deprecation guide explain how to keep using the broccoli pipeline whilst they migrate (using `/@embroider/virtual/app.css`)

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

None that I can think of.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

Add a guide to the docs on how to opt in to vite's CSS pipeline.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
