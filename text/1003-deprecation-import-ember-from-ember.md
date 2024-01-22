---
stage: accepted
start-date: 2024-01-22T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1003 
project-link:
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
-->

<-- Replace "RFC title" with the title of your RFC -->
# Deprecate `import Ember from 'ember'; 

## Summary

This RFC proprosing deprecating all APIs that have module-based replacements, as described in [RFC #176](https://rfcs.emberjs.com/id/0176-javascript-module-api) as well as other `Ember.*` apis that are no longer needed.

## Motivation

> Why are we doing this? What are the problems with the deprecated feature?
What is the replacement functionality?

## Transition Path


### `Ember.testing`

Instead, use

```js
import { macroCondition, isTesting } from '@embroider/macros';

// ...

if (macroCondition(isTesting()) {
  // test only code here
}
```

ENV/EmberENV ... equivalent shape thing from config/environment

Ember.lookup (what is this?)

### `Ember.onError`

```js
window.addEventListener('error', /* ... event handler ... */);
```

### `Ember.BOOTED`

No replacement. Unused.

### `Ember.TEMPLATES`

This is the template registry. It does not need to be public API. 

No replacement.

### Ember.onload (has replacement) ... what is runloadHooks?

### Ember.HTMLBars / Ember.Handlebars (must go)

### Ember.Test/Ember.setupForTesting (should be replaced by @ember/test-helpers??)

### Container

### Registry

### _setComponentManager (has replacement on @ember/component)

### _modifierManagerCapabilities (has replacement on @ember/modifier)

### _componentManagerCapabilities

### meta (tagged private but exported on 'ember')

### RSVP

### libraries (may need public replacement)

### ActionHandler (no public API)

### Comparable (used by import compare from @ember/utils)

## How We Teach This

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?
Does it mean we need to put effort into highlighting the replacement
functionality more? What should we do about documentation, in the guides
related to this feature?
How should this deprecation be introduced and explained to existing Ember
users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.
There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
