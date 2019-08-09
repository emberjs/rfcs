- Start Date: 2018-09-21
- RFC PR: [https://github.com/emberjs/rfcs/pull/380](https://github.com/emberjs/rfcs/pull/380)
- Ember Issue: (leave this empty)

# Add `queryParams` to the router service

## Summary

This RFC proposes a new primitive API change to the `RouterService` to allow access to query params from anywhere.

Access to query params is currently restricted to the controller, and subsequently, the corresponding route.
This results in some limitations with which users may consume query param data.
By exposing query params on the [RouterService](https://api.emberjs.com/ember/release/classes/RouterService), users will be able to easily access the query params from deep down a component tree, removing the need to pass query param related data and actions down many layers of components.



## Motivation

Modern SPA concepts have converged on the idea that query params should be easily accessible -- independent from the object responsible for handling the route.
Like with the [RouterService](https://github.com/emberjs/rfcs/blob/master/text/0095-router-service.md),
it is common to have a need to perform routing behavior from deep down a component tree.
Additionally, the current query params implementation feels very verbose for "just wanting to access a property" and has been frustrating to have to explain awkward behavior when on-boarding new devs who may be unfamiliar with Ember.

Accessing data within the url **should feel easy**.

**What's wrong with the existing query params?**
- For all but one use case, controllers can be avoided. Query Params force controllers into existence for those who are trying to avoid them. 
- The caching mechanism is persistent beyond just child routes. _Any_ time a route with query params is re-visited, the query params values will be restored. This can be useful for those who are needing this behavior, but for those who want to manage queryParams via transition / navigations, they'd need to set up query param resets on enter/exit of routes, link-to's and transitions -- the query params implementation becomes more of an obstacle than a feature.

## Detailed Design

### Accessing Query Params

```ts
import Component from "@glimmer/component";
import { inject as service } from '@ember/service';

export default class Pagination extends Component {
  @service router;

  get currentPage() {
    const { page } = this.router.queryParams;

    return page;
  }
}
```

Having query params accessible on the router service would allow users to implement:

 - query param aware modals that may hide or show depending on the presence of a query param.
 - fill in form fields from a link.
 - filter / search components could update the query param property.
 - whatever else query params are used for outside of a SPA.

### Serialization / Deserialization

Everyone has different query param serialization and deserialization needs depending on a variety of factors.

By default, the query params will be serialized and deserialized via the builtin URLSearchParams API, and polyfilled for IE11.

Should someone decide to customize how serilaization and deserializion transforms the query params, that can be done directly on the router:

```ts
import EmberRouter from '@ember/routing/router';

import config from '../config/environment';

const Router = EmberRouter.extend({
  location: config.locationType,
  rootURL: config.rootURL,

  queryParamsConfig: {
    serialize(queryParams: object): string {
      // serialize object for query string
      // default to URLSearchParams, polyfilled for IE11
    },
    deserialize(queryString: string): object {
      // parse to object for use in `injectedRouter.queryString`
      // also default to URLSerachParams
    }
  }
});
```


This will address a long standing issues from as far back as 2016,
some new functionality for serialization and deserialization could be powered by [qs](https://www.npmjs.com/package/qs) ([3.4kb (gzip+min)](https://bundlephobia.com/result?p=qs@6.7.0)) or a lternatively, [URLSearchParams](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams) -- this would enable the setting of arrays and objects which is not possible today.


### Sticky Query Params

By default, transitionTo will clear the query params, unless specified inside the transiion.

If query params are defined ahead of time as sticky, they will persist in the URL between sub routes.

This can be configured in the `Router.map` function:
```ts
Router.map(function() {
  this.queryParams('bar');

  this.route('faq');

  this.route('posts', function() {
    this.queryParams('foo', 'baz');

    this.route('new');
    this.route('index');
    this.route(
      'show',
      { path: '/:postId', queryParams: ['hideComments', 'invertColors'] },
      function() {
        this.route('edit');
        this.route('comment');
        this.route('share');
      }
    );
  });
});
```

This method of dealing with query params implies the following:
 - All query params, even if not specified are allowed. given the above example, I could use the qp "strongestAvenger" anywhere
 - Any sort of transition will clear the query params, unless it is defined as a sticky queryParam. So, if I'm on the posts/new route with the query params "foo" and "baz", and transition to posts/show, those query params are still available. If I navigate to the faq page, foo and baz are cleared from the URL.
 - The globally defined query params will stick around until cleared manually. If I visit faq?bar=2, and then transition to posts. the bar=2 query param will still be present.
 - The `this.queryParams` function will be available at every nesting of the route tree, but `queryParams` are also configurable in the route options hash.

Notes:
 - This does not imply a caching mechanism (like what controllers today allow)
 - If a certain app wants caching of query params as they exist today, an addon could be made to fill that gap.

 **A more concrete example: filter query params**
Given we want a way to search over a list of products, be able to view additional information about a selected search result, and save the search result for later, sticky params help maintain the search throughout all of the route navigations (similar to Amazon). 


```ts
Router.map(function() {
  this.queryParams('bar');

  this.route('search', { queryParams: [
    'term', 'isPrime', 'department', 'averageReview', 'brand', 
    'memoryType', 'processor', 'vRamCapacity', 'certification',
  ]}, function() {
    // index route will be the search results page

    // shows a selected result with additional information
    this.route('summary', { path: '/summary/:itemId' });

    // shows a modal with a field to name the search to be loaded later
    this.route('save');
  });
});
```

1. visit `/search`.
2. type "RTX" into the search bar press the enter key.
3. the URL now shows `/search?term=RTX` and some results display as rows.
   The code for appending `term` may look like the following:

   ```ts
   @service router;

   this.router.setQueryParam('term', 'RTX');
   ```

4. maybe there is a side panel that has dynamically updated with relevant 
   potential search values, you check that you want "Prime" only results.
5. the URL now shows `/search?term=RTX&isPrime=true`.
6. clicking on the first result expands the row to fill more of the screen.
   The transition to the new route may look like the following:

   ```ts
   this.transitionTo('search.summary', '001');
   ```
7. the URL is now `/search/summary/001?term=RTX&isPrime=true`.
   the query params are still present 
   because the parent route defined sticky query params.
8. clicking "close" on the expanded row transitions back to `/search?term=RTX&isPrime=true`

   ```ts
   this.transitionTo('search');
   ```
9. clicking "save search" opens a modal where you may save your search results.
   implementation of the submit action make look like:

   ```ts
   @service router;

   @action submit() {
     let { queryParams } = this.router

     await fetch('some-url', {
       method: 'POST',
       body: JSON.stringify({
         name: this.name,
         search: queryParams
       }),
     });
   }

   ```


-------------------------------------------

The biggest change needed, which could _potentially_ be a breaking change, is that the allow-list on routes' queryParams hash will need to be removed. The controller-based way is static in that all known query params must be specified on the controller ahead of time. This has been a great deal of frustration amongst many developers, both new and old to ember.
This is a dynamic way to manage query params which hopefully aligns more with people's mental model of query params.

### Setting Query Params

Until IE11 support is dropped, we cannot wrap and set query params intuitively as a normal getter/setter as is proposed by this addon.

For example, this is not possible until IE11 support is dropped:

```ts
this.router.queryParams.strongestAvenger = 'Hulk';
```

This is due to the fact that IE11 only supports ES5, which [does not have Proxy support](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy#Browser_compatibility) and [cannot be polyfilled](https://babeljs.io/docs/en/learn#proxies).

> Due to the limitations of ES5, Proxies cannot be transpiled or polyfilled.

- _[https://babeljs.io/docs/en/learn#proxies](https://babeljs.io/docs/en/learn#proxies)


Instead, functions must be defined on the router that would take care of the setting of the query param's value.
```ts
// set a single parameter
this.router.setQueryParam('strongestAvenger', 'Captain Marvel');
// set many parameters
// this'll replace
this.router.setQueryParams({
  strongestAvenger: 'Carol Danvers',
  secondStrongest: 'Thor or Hulk?',
  ['kebab-case-param']: `kebab o' Thanos' armies`,
});
```

Just as it is today, setting query params in this way would not trigger a transition.
The key-value state of set query params would be available on `<RouterService>#queryParams` as well as an existing controller's computed properties as they exist today.

### Interop with Today's Query Params

Controllers will still need to have their queryParams explicitly set, but for maximum ineteropability:

**Setting a query param with the new API**
```ts
this.router.setQueryParam('key', 'value');
```
The current route is known, and the hierarchy of active controllers are also known. `setQueryParam` could look through the controllers to see if any of them define a query param `key` and then set it.

**Getting a query param with the new API**
```ts
this.router.queryParams.key;
```
Simalar to setting the query param, 
the getter could also look through the hierarchy of active controllers 
to see if a value exists. 

Accessing or Setting query params using the new APIs that happen to collide 
with controller-defined query params should throw a warning, 
as the interop would give the router's query params an implicit cache, 
which may not be present in a future without controllers.

**Setting a query param with the old API**
```ts
this.set('controllerQp', 'value');
```
It's possible to have `set` use `this.router.setQueryParam`, 
but it would need to have additional code defined in `set` 
which checks if the current instance is a `Controller` 
and if the `queryParams` array defines the key. 

**Getting a query param with the old API**
Because this is only possible with query params defined on controllers, 
the value of the query param *must* exist on the controller. 
There is no need to have compatibility with the router's queryParams here.

## How we teach this

Currently, query params _must_ be [specified on the controller](https://guides.emberjs.com/release/routing/query-params/):
```ts
export default class extends Controller {
  queryParams = ['page', 'filter', {
   // QP 'articles_category' is mapped to 'category' in our route and controller
   category: 'articles_category'
  }];
  category = null;
  page = 1;
  filter = 'recent';

  @computed('category', 'model')
  get filteredArticles() {
    // do something with category and model as category changes
  }
}
```

Having query-param-related computed properties available everywhere will be a shift in thinking that "the controller manages query params" to "query params are a routing concern and are on the router service"

```ts
import Route from '@ember/routing/route';
import { inject as service } from '@ember/service';
import { alias } from 'ember-query-params-service';

export default class ApplicationRoute extends Route {
  @service router;

  @alias('router.queryParams.r') isSpeakerNotes;
  @alias('router.queryParams.slide') slideNumber;

  model() {
    return {
      isSpeakerNotes: this.isSpeakerNotes,
      slideNumber: this.slideNumber
    }
  }
}
```

## Drawbacks

- Some people may be relying on the controller query-params allow-list.
- Some people may be super tied in to controller query params cacheing.
