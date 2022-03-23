---
Stage: Accepted
Start Date: 
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember CLI, Learning
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

# Add `ember-modifier` dependency

## Summary

This RFC proposes adding [ember-modifier](https://github.com/ember-modifier/ember-modifier)
to the blueprints that back `ember new` and `ember addon`.

This RFC supersedes the original [RFC #353 "Element Modifiers"](https://github.com/emberjs/rfcs/pull/353).

## Motivation

[RFC #373 "Element Modifier Manager"](https://emberjs.github.io/rfcs/0373-Element-Modifier-Managers.html)
introduced low-level primitive for defining element modifiers.
Addon `ember-modifier` was built on top of those low-level primitives and provides
convenient API for authoring element modifiers in Ember.

Element Modifiers introduced as part of Ember Octane edition and first class citizens
in Ember application, like components or helpers. However today, developers need to install
additional library to be able to define custom modifier.

THis RFC seeks to fill the gap in Ember.js development mental model with
providing `ember-modifier` in the blueprint.

> Why are we doing this? What use cases does it support? What is the expected
outcome?

## Detailed design

The necessary changes to `ember-cli` are relatively small since we only need
to add the dependency to the `app` blueprint, and the `addon` blueprint will
inherit it automatically.

This has the advantage (over including it as an implicit dependency), that
apps and addons that don't want to use it for some reason can opt out by
removing the dependency from their `package.json` file.

## How we teach this

Ember Guides already teach how to use modifiers in "Template Lifecycle, DOM, and Modifiers"
section of the guides.

The only change we should make is to remove "install `ember-modifier`" instruction
from the prose as that dependency will be part of the blueprint.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

- Introduce [RFC #757 "Default Modifier Manager](https://github.com/emberjs/rfcs/pull/757)
  as default way creating modifiers and ask developers to install `ember-modifier`
  for more complex use cases.

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

None.