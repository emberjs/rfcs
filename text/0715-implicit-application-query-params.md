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

Goals:
 - the set of query param values accessible on the currentRoute are a direct result
   of the URL
 - query param values are never out of sync.
 - with the optional feature flag enabled, controllers do not set values as a
   result of URL transitions

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

### Examples

Most "fancy features" of query params would be implemented in user-space or in
supplemental addons, such as ember-parachute.

#### Sticky Query Params

[Demo Ember Twiddle](https://ember-twiddle.com/7e472191b3f5021433b8552158a4379e?openFiles=routes.articles%5C.js%2C&route=%2Farticles)

> Note that this example has been implemented in Ember 3.18, and would be
> significantly simpler given the optional feature proposed in this RFC.

Sticky query params are supported by controllers by default, but with the optional
feature flag enabled, there is no sticky behavior. Query params are strictly a reflection
of what is in the URL. To re-implement sticky query params, an addon might have an
implementation similar to this (and/or the twiddle).

```ts
// app/services/query-params.ts
const CACHE = new Map<Record<string, string>>();

function getForUrl(url: string) {
  let existing = CACHE.get(url);

  if (!existing) {
    CACHE.set(url, {});

    return existing;
  }

  return existing;
}

class QueryParamsService extends Service {
  @service router;

  setQP(qpName, value) {
    let cacheForUrl = getForUrl(this.router.currentURL);

    cacheForUrl[qpName] = value;
  }

  getQP(qpName) {
    return getForUrl(this.router.currentURL)[qpName];
  }
}
```

```js
// some route
export default MyRoute extends Route {
  @service router;
  @service queryParams;

  async beforeModel({ to: { queryParams }}) {
    let category = this.queryParams.getQP('category');

    // if transitioning to a route with explicit query params,
    // update the query param cache
    if (stickyQPs.category && category !== queryParams.category) {
      this.queryParams.setQP('category', queryParams.category);
    }

    // because the URL must reflect the state query params, we must transition
    if (!queryParams.category && category) {
      this.router.transitionTo({ queryParams: { category }});
    }
  }
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
