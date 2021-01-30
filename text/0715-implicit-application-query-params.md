---
Stage: Accepted
Start Date: 2020-01-30
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/715
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

# Implicit Application Query Params

## Summary

This RFC doesn't aim to change existing behavior but does aim to allow opting
developers to use *any* Query Param passed to `transitionTo`. This will hopefully
improve the ergonomics of using query params in most default situations.

## Motivation

Query Params currently require a bit of ceremony to be used at all. If Query
Params today opted you out of the current restrictive-query-params implementation
then the default experience with query params would "just work", improving the
learning experience as well for new developers once they learn about `transitionTo`.


## Detailed design

This may require an optional feature to be non-breaking for folks that rely on
forbidding all query params by default.

Query Params as they work today are allow-list only, and block all other Query Params.

Example:
```js
export default class ArticlesController extends Controller {
  queryParams = ['category'];
}
```

With `queryParams` specified, only `category` will be allowed on the route for
this controller.

This RFC only proposes a change only to this scenario:
```js
// Empty or default controller
export default class ArticlesController extends Controller {
}
```

where now `<RouterService>#transitionTo({ queryParams: { category: 'anything' } })`
and LinkTo will _not_ strip away the `category` Query Param.

The way this would work is that _all_ Query Params detected during a transition
would live on the `ApplicationController`, as if the `queryParams` array somehow
knew of all possible Query Params.


## How we teach this

The current guides would need updating teach that query params could be used without
modifying controllers. While the existing Query Params behavior would still work,
and provide much more utility than lumping everything on the `ApplicationController`
implicitly, it would be more of an "Advanced Topic".

## Drawbacks

- Updating the guides would probably be easier if done in conjunction with
[RFC#712: Query Params as derived state](https://github.com/emberjs/rfcs/pull/712)

- Adding an optional feature increases test matrices and adds overall complexity
  to the routing system, but changing the default behavior of `queryParams = []`
  (or unspecified) could break people's apps.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
