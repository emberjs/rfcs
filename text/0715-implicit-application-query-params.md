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

# Arbitrary Query Params

## Summary

This RFC aims to add opt-in behavior that would allow developers to use *any*
Query Param passed to `transitionTo`. This will hopefully improve the ergonomics
of using query params in most default situations.

## Motivation

Query Params require configuration to use today, which is counter to our
"convention over configuration" philosophy. This RFC proposes an opt-in change to
allow for query params to _always_ be used in transitions and links, and no longer
require the allow-list that controllers currently implement today.

## Detailed design

Arbitrary Query Params will need to be guarded by an optional feature, because:
 - controllers are allow-list only, and the ability to allow all query params
   by default conflicts with the feature set of a controller's allow-list.
 - query param values will not be stored on any controller, whereas the
   controller is the only place tracked-access to query params exists.


### Example

Query Params as they work today are allow-list only, and block all other Query Params.

Example:
```js
export default class ArticlesController extends Controller {
  queryParams = ['category'];
}
```

With `queryParams` specified, only `category` will be allowed on the route for
this controller.

This RFC only proposes a change for this scenario:
```js
// Empty or default controller
export default class ArticlesController extends Controller {
}
```

where `<RouterService>#transitionTo({ queryParams: { category: 'anything' } })`
and LinkTo will _not_ strip away the `category` Query Param.

### Migration Path

1. Update optional-features.json

    ```
    // config/optional-features.json
    {
      "arbitrary-query-params": true
    }
    ```

2. Delete Query Param allow lists

    ```diff
     export default class ArticlesController extends Controller {
    -  queryParams = ['category'];
     }

    ```

## How we teach this

The current guides would need updating teach that query params could be used without
modifying controllers. While the existing Query Params behavior would still work,
and provide much more utility than always arbitrary query params, it would be more
of an "Advanced Topic".


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
