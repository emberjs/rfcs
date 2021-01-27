---
Stage: Accepted
Start Date: 2020-01-26
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/712
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

# Query Params as derived data

## Summary

Query Params are awkward in ember in that they align more to the paradigms of a much older
ember. In the spirit of Octane, this RFC proposes to update how we think about query params
such that they align with the idea that they are derived state from the URL, much like
native getters have become in components / services / etc

## Motivation

As taught in [the current version of the guides (3.24)](https://guides.emberjs.com/v3.24.0/routing/query-params/),

Query Params do not currently follow the spirit of Octane:
 - All data is derived from some source of data
 - Unidirectional data flow, eliminating "spooky action from a distance"

> All data is derived from some source of data

The source of the data when it comes to Query Params is the URL. With the current way
Query Params are taught, there are two sources: the URL, and the (sometimes @tracked) property
on the controller.

```js
export default class ArticlesController extends Controller {
  queryParams = ['category'];

  // looks like it could be the source of data, but is also "spookily" overwritten during transitions
  category = null;
}
```

> Unidirectional data flow, eliminating "spooky action from a distance"

There are several ways to update both a query param and the URL.
With the above example, `category` is _two way bound_ to the URL -- the URL updates `category`,
and updates to `category` cause transitions to the URL.

Explicit transitions can cause the URL to change, which then cause `category` to change as well.


**Proposal**: we teach a more direct way of interacting with query params via "derived data".

Example,

```js
import Controller from '@ember/controller';
import { action } from '@ember/object';
import { inject as service } from '@ember/service';

export default class ArticlesController extends Controller {
  @service router;

  queryParams = ['category'];

  get categoryFromQueryParams() {
    return this.router.currentRoute.queryParams.category;
  }

  @action
  updateCategory(value) {
    this.router.transitionTo({ queryParams: { category: value } });
  }
}
```

The biggest advantage here is that interacting with query params between all class types
is _consistent_. Changing a query param from a controller, route, component, service, custom class, etc
can change and access the query params the exact same way. No need to force prop-drilling patterns
by passing controller properties down many layers of components.

For example, assume there is a compnoent 10+ component layers deep. Instead of passing `@category` and
`@updateCategory` through each and every one of those component layers, the component can do do:

```js
import Component from '@glimmer/controller';
import { action } from '@ember/object';
import { inject as service } from '@ember/service';

export default class MyDeeplyNestedComponent extends Component {
  @service router;

  get category() {
    return this.router.currentRoute.queryParams.category;
  }

  @action
  updateCategory(value) {
    this.router.transitionTo({ queryParams: { category: value } });
  }
}
```


## Detailed design

This pattern already works so all that would need updating are the Guides.

## How we teach this

TODO

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
