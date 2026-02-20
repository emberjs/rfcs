---
stage: accepted
start-date: 2023-08-18T00:00:00.000Z 
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: 
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/946
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

# Unified imports for `<template>` 

## Summary

> One paragraph explanation of the feature.

Ported from: https://github.com/emberjs/rfcs/issues/945


I've been trying out [`ember-template-imports`](https://github.com/ember-template-imports/ember-template-imports) in my app, really liking this new way of defining components so far. Thanks to everyone who made it happen.

One thing that's been bothering me slightly is the number of imports that some of my components need. I'm sure that better editor support will help, as will a possible future consolidation of core Ember imports, but I wish importing common things wasn't so scattered and verbose, eg:

```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { service } from '@ember/service';
import { on } from '@ember/modifier';
import { array, hash, fn } from '@ember/helper';

export default class extends Component {
  @tracked isVisible = false;

  <template>
    ...
  </template>
}
```

I think there's an opportunity to simplify these imports. 

In my own app, I've stared to re-export common built-ins:

**`<my-app>/index.js`**
```js
export { default as Component } from '@glimmer/component';
export { tracked } from '@glimmer/tracking';
export { service } from '@ember/service';
export { on } from '@ember/modifier';
export { array, hash, fn } from '@ember/helper';
```

Which makes importing much simpler:

**`<my-app>/components/modal.gjs`**
```js
import { Component, on, or, tracked } from 'vidu-web';

export default class extends Component {
  @tracked isVisible = false;

  <template>
    ...
  </template>
}
```

Perhaps we should make a change in the framework to make importing common built-ins simpler? A few possibilities:

**1. From `@ember`**:
```js
import { Component, on, or, tracked } from '@ember';
```

**2. From `@ember/<edition>`**:
```js
import { Component, on, or, tracked } from '@ember/polaris';
```

**3. From `<my-app>`**: (we'd change the blueprints to do the re-export)
```js
import { Component, on, or, tracked } from '<my-app>';
```


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
