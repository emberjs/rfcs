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

For example, assume there is a component 10+ component layers deep. Instead of passing `@category` and
`@updateCategory` through each and every one of those component layers, the component can do:

```js
import Component from '@glimmer/component';
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

These changes will need to be made to the guides:

### Query Parameters / Intro

Unchanged

### Specifying Query Parameters

Query params are declared on route-driven controllers. For example, to
configure query params that are active within the `articles` route,
they must be declared on `controller:articles`.

To add a `category`
query parameter that will filter out all the articles that haven't
been categorized as popular we'd specify `'category'`
as one of `controller:articles`'s `queryParams`:

```javascript {data-filename=app/controllers/articles.js}
import Controller from '@ember/controller';
import { inject as service } from '@ember/service';

export default class ArticlesController extends Controller {
  queryParams = ['category'];

  @service router;

  get category() {
    return this.router.currentRoute.queryParams.category;
  }
}
```

This sets up an allow list of query params on that only permits the `category`
query param in the URL, In other words, no other query params are allowed
on `controller:articles`. Any changes to the `category` query param in the URL
will update the query params on the `currentRoute` of the `router` which will
then be reflected in the `category` getter on the controller.

Note that you can't make `queryParams` be a dynamically generated property
(neither computed property, nor property getter); they have to be values.

Also Note that to change the URL programatically,
`this.router.transitionTo({ queryParams: { category: 'next category' } })`
can be used.

Now we need to define a getter for our category-filtered
array, which the `articles` template will render.

```javascript {data-filename=app/controllers/articles.js}
import Controller from '@ember/controller';
import { inject as service } from '@ember/service';
import { tracked } from '@glimmer/tracking';

export default class ArticlesController extends Controller {
  queryParams = ['category'];

  @service router;

  get category() {
    return this.router.currentRoute.queryParams.category;
  }

  get filteredArticles() {
    let category = this.category;
    let articles = this.model;

    if (category) {
      return articles.filterBy('category', category);
    } else {
      return articles;
    }
  }
}
```

With this code, we have established the following behaviors:

1. If the user navigates to `/articles`, `category` will be `undefined`, so
   the articles won't be filtered.
2. If the user navigates to `/articles?category=recent`,
   `category` will be set to `"recent"`, so articles will be filtered.
3. Once inside the `articles` route, `transitionTo` may be used to change
   the `category` query param. By default, a query param property change won't
   cause a full router transition (i.e. it won't call `model` hooks and
   `setupController`, etc.); it will only update the URL.


### <LinkTo /> component

TODO

### transitionTo

TODO

### Opting in to a full transition

TODO

### Update URL with 'replaceState' instead

TODO

### Map a controller's property to a different query param key

Removed

### Default values and (de)serialization

TODO

### Sticky Query Param Values

------------------------------------


Questions from [RFC #715](https://github.com/emberjs/rfcs/pull/715), which takes
this (#712) RFC a step further and removes the reliance on controllers for query
params:

Most "fancy features" of query params would be implemented in user-space:

### Sticky Query Params

Sticky query params are supported by controllers by default, but but if someone
wanted to manage that state themselves (for additional features, or providing
different behavior), they may be able to implement it like this:

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

### Default Query Params

Controllers also support default query params, but encourage the use of a
query param property, which is two-way-bound. To have a one-way dataflow
from the URL, default query params may look like this:

```js
export default MyRoute extends Route {
  @service router;

  async beforeModel({ to: { queryParams }}) {
    if (!queryParams.category) {
      this.router.transitionTo({ queryParams: { category: 'default value' }});
    }
  }
}
```

### Does <Route>#refrosh() retain query params?

yes

### Does the route model hook receive query params?

yes, on `transition.to.queryParams`

### How could sticky query params in links handled in user-space?

If sticky query params are managed on a service, a custom `LinkTo` component could
inject that service and look up if query params have been set for a particular
route target -- and the implementation could decide if the query params should be
sticky with or without the dynamic segments. A primitive implementation that uses
the full href + dynamic segments might look like:

```ts
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

  @action
  cacheQP(qpName, value) {
    let cacheForUrl = getForUrl(this.router.currentURL);

    cacheForUrl[qpName] = value;
  }

  @action
  getQP(qpName) {
    return getForUrl(this.router.currentURL)[qpName];
  }

  @action
  forUrl(url: string) {
    return getForUrl(url);
  }
}
```

```ts
// your custom link component
interface Args {
  href: string;
}

export default class MyLink extends Component<Args> {
  @service router;
  @service queryParams;

  get href() {
    // or this.router.urlFor(...);
    let { href } = this.args;
    let qps = stickyQPsToQueryString(this.queryParams.forUrl(href));

    return `${href}?${qps}`;
  }
}

function stickyQPsToQueryString(queryParams: Record<string, string>) {
  let search = new URLSearchParams();

  for (let qp in queryParams) {
    let value = queryParams[qp];

    search.add(qp, value);
  }

  return search.toString();
}
```
```hbs
<a href={{this.href}} ...attributes>{{yield}}</a>
```


## Drawbacks

This is a major change in how we think about query params and could be jarring
for folks, and could take some getting used to.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
