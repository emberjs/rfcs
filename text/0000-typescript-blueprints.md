---
Stage: Accepted
Start Date: 2021-11-11
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
  Relevant Team(s): Ember CLI
RFC PR:
---

<!---
Directions for above:

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Author Built-In Blueprints in TypeScript

## Summary

> One paragraph explanation of the feature.

## Motivation

Ember CLI's generators are a foundational part of Ember itself. They are one of the first elements of the developer experience that Ember prides itself on, and continue to be an essential part of the ecosystem by virtue of the fact that both apps and addons can generate their own custom blueprints. For many years, the `ember-cli-typescript` addon has shipped its own set of TypeScript-flavored built-in blueprints that essentially act as drop-in replacement for the built-in blueprints that ship with Ember today. However, these blueprints have proven very difficult to maintain as it involves keeping them all up to date with every single change made to Ember itself, so much so that the current TypeScript blueprints have diverged to the point there is currently not a TypeScript blueprint for components at all.

As part of the larger effort to [make Typescript an officially supported language in Ember](https://github.com/emberjs/rfcs/pull/724), we should ship TypeScript blueprints in Ember itself, in place of the existing Javascript blueprints. This would provide several benefits over the current system:

1. Remove the disconnect between TypeScript and JavaScript blueprints discussed above, as the `ember-cli-typescript` team would no longer be responsible for maintaining an entirely independent set of blueprints.
1. Pave the way for TypeScript blueprint generation in any Ember app or addon right out of the box (likely via a `--typescript` flag or something similar).
1. Enable addon authors to better support TypeScript users by allowing them to publish their own TypeScript blueprints without concern for also supporting JavaScript users.

Note that this does _not_ mean that we should _generate_ TypeScript by default, but rather that the blueprints themselves should _begin_ as TypeScript before actually being reified as actual code. By default, all of the Ember generators would still behave exactly as they do today: by generating Javascript files. This RFC is primarily concerned with TypeScript as an _implementation detail_ for the existing blueprints, rather than being prescriptive about whether or not users should adopt TypeScript themselves.

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
> familiar with the framework to understand, and for somebody familiar with the
> implementation to implement. This should get into specifics and corner-cases,
> and include examples of how the feature is used. Any new terminology should be
> defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
> idea best presented? As a continuation of existing Ember patterns, or as a
> wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
> re-organized or altered? Does it change how Ember is taught to new users
> at any level?

> How should this feature be introduced and taught to existing Ember
> users?

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
