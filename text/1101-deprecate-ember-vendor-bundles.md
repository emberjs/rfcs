---
stage: released
start-date: 2025-05-13T00:00:00.000Z
release-date:
release-versions:
  ember-source: 6.10.0
teams:
  - cli
  - framework
  - learning
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1101'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1156'
  released: 'https://github.com/emberjs/rfcs/pull/1159'
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

# Deprecate Ember Vendor Bundles

## Summary

The published `ember-source` package contains several AMD-specific bundled builds
of Ember that are appended to `vendor.js` in the classic build system. 
This RFC deprecates the following files:

- `ember.debug.js`
- `ember.prod.js`
- `ember-testing.js`
- `ember-template-compiler.js`

Instead, Ember will be included in application builds as ES module library via `ember-auto-import`.

## Motivation

Maintaining the legacy AMD-specific bundled copies of Ember is no longer necessary. 
Modern Ember applications should consume Ember as ES modules, which aligns with 
the broader JavaScript ecosystem. This change simplifies the build pipeline and 
reduces maintenance overhead.

This deprecation will have no effect on applications using Embroider with 
`staticEmberSource: true` or Embroider v4 (Vite). It only impacts applications 
using the classic build system without Embroider.

## Transition Path

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
> familiar with the framework to understand, and for somebody familiar with the
> implementation to implement. This should get into specifics and corner-cases,
> and include examples of how the feature is used. Any new terminology should be
> defined here.

> Please keep in mind any implications within the Ember ecosystem, such as:
>
> - Lint rules (ember-template-lint, eslint-plugin-ember) that should be added, modified or removed
> - Features that are replaced or made obsolete by this feature and should eventually be deprecated
> - Ember Inspector and debuggability
> - Server-side Rendering
> - Ember Engines
> - The Addon Ecosystem
> - IDE Support
> - Blueprints that should be added or modified


### Classic Build System

We will introduce a new optional feature for projects to opt into consuming 
Ember as ES modules. Not having this optional feature enabled will result in 
a deprecation warning.

Addons that rely on accessing Ember from `treeForVendor` or on accessing Ember
from vendor will need to update their implementation. 

### Embroider v4 or with `staticEmberSource: true`

This deprecation will have no effect on these projects. They already consume Ember as ES modules.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
> idea best presented? As a continuation of existing Ember patterns, or as a
> wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
> re-organized or altered? Does it change how Ember is taught to new users
> at any level?

> How should this feature be introduced and taught to existing Ember
> users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

### Deprecation Guide

Ember will no longer publish legacy AMD-specific Ember builds. To opt-in to 
consuming Ember as ES modules and clear this deprecation, enable the 
`ember-modules` optional feature by running `ember feature:enable ember-modules`.

This applies only to the classic build system or to Embroider < 4.0 without the
`staticEmberSource: true` option. If you see this deprecation warning in these 
setups, please [open an issue](https://github.com/emberjs/ember.js/issues/new/choose).

Alternatively, you can also clear the deprecation by moving to Embroider v4 by 
running the [Ember Vite Codemod](https://github.com/mainmatter/ember-vite-codemod),
but this may require additional changes to your project. 

The AMD-specific Ember builds will no longer be published in next Ember major release 
and no longer be bundled into `vendor.js`, even on the classic build system. These files are:
- 
- `ember.debug.js`
- `ember.prod.js`
- `ember-testing.js`
- `ember-template-compiler.js`

- In rare cases, Addons were relying on accessing Ember from `vendor`. If you have 
addons that do so they will need to be updated to consume Ember as ES modules.

A known addon that previously relied on accessing Ember from `vendor` is 
[ember-cli-deprecation-workflow](https://github.com/ember-cli/ember-cli-deprecation-workflow). 
Please ensure you are on the latest version of this addon as that reliance has 
been removed.

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Ember,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
> TBD?
