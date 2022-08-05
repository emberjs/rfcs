---
start-date: 2021-06-10T00:00:00.000Z
release-date:
release-versions: 
teams: 
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/752
project-link: 
stage: accepted
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

# Rename `inject` import to `service`

## Summary

Currently, in order to use a service in any framework class you can do it like this:

```js
import { inject } from '@ember/service';

export class MyComponent extends Component {
  @inject router;
}
```

However, it is very common to actually alias this import to `service`, like this:

```js
import { inject as service } from '@ember/service';

export class MyComponent extends Component {
  @service router;
}
```

This RFC proposes to actually provide this as a `service` import directly.


## Motivation

You cannot properly/easily use editor autocompletion to import `@service`. 

Even in the guides, `inject` is aliased to `service` (see: https://guides.emberjs.com/release/services/#toc_accessing-services)

Finally, there is also an `inject` export from `@ember/controller`, which is also a possible source of confusion.

By exporting this directly as `service`, this would be streamlined:

```js
import { service } from '@ember/service';

export class MyComponent extends Component {
  @service router;
}
```

## Detailed design

We add an export `service` from the `@ember/service` package with the semantics
of `inject`.

Import of `inject` itself will be deprecated targeting Ember 5.0 for removal,
and the app blueprint will be updated as soon as possible to the new API.

Addons will likely want to support the legacy API for a long time, especially
if the legacy API is not deprecated in Ember 4.0.0. The addon blueprint and
best practice should be to:

```js
import * as ES from '@ember/service';
const service = ES.service ?? ES.inject;
```

Then use `service` for injections.

## How we teach this

The docs should be updated to directly import `import { service } from '@ember/service';`.

## Drawbacks

It might be considered churn. However, we could probably provide a codemod to
automatically rename the imports, lessening the churn.

Addons will need to live with a degraded injection API for a long time, if they
want to support both versions.

## Alternatives

We could leave the import as it is, continuing to suggest people rename the import to `service`. 

Or we could stop suggesting the import and push for the community to simply use `@inject myService`.

We could also choose a different name for the import (e.g. `injectService` or something like this).

## Unresolved questions
