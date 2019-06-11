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

### Transitioning to Add and Remove Query Params

Given there is a route that exists named `avengers.roster`,

 - Adding a query param
    ```ts
    import Component from "@glimmer/component";
    import { inject as service } from '@ember/service';
    import { action } from '@ember/object';

    export default class Pagination extends Component {
      @service router;

      @action nextPage() {
        // `this.router.queryParams` is currently an empty object
        const nextPage = (this.router.queryParams.page || 0) + 1

        this.router.transitionTo('avengers.roster', { queryParams: { page: nextPage } });
        // after the transition, `this.router.queryParams` is `{ page: 1 }`
      }
    }
    ```

 - Removing all query params
    ```ts
    import Component from "@glimmer/component";
    import { inject as service } from '@ember/service';
    import { action } from '@ember/object';

    export default class Pagination extends Component {
      @service router;

      @action reset() {
        // `this.router.queryParams` is currently `{ page: 1 }`

        this.router.transitionTo('avengers.roster', { queryParams: { /* empty */ } });
        // after the transition, `this.router.queryParams` is `{ }`
      }
    }
    ```

 - Merging query params

    We have previously been to the route `avengers.roster` with query params `{ page: 1 }`
    ```ts
    import Component from "@glimmer/component";
    import { inject as service } from '@ember/service';
    import { action } from '@ember/object';

    export default class HeaderNavigation extends Component {
      @service router;

      @action navigateToRoster() {
        // `this.router.currentRouteName` is currently `'index'`
        // `this.router.queryParams` is currently `{ }`

        this.router.transitionTo(`avengers.roster`);
        // after the transition, `this.router.queryParams` has restored the value of `{ page: 1 }`
        // from the controller state
      }
    }
    ```


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
this.router.setQueryParameter('strongestAvenger', 'Captain Marvel');
// set many parameters
// this'll replace 
this.router.setQueryParameters({
  strongestAvenger: 'Carol Danvers',
  secondStrongest: 'Thor or Hulk?',
  ['kebab-case-param']: `kebab o' Thanos' armies`,
});
```

Just as it is today, setting query params in this way would not trigger a transition.
The key-value state of set query params would be available on `<RouterService>#queryParams` as well as an existing controller's computed properties as they exist today.

This doesn't change the existing functionality where objects and arrays are still not correctly serialized to their URI-string counterparts.

To address that, a long standing issue from as far back as 2016,
some new functionality for serialization and deserialization would be powered by [qs](https://www.npmjs.com/package/qs) ([3.4kb (gzip+min)](https://bundlephobia.com/result?p=qs@6.7.0)) -- which would enable the setting of arrays and objects.


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

This will no longer be true. However, controllers will still manage the long-term state of the query params as they do today.

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

## Alternatives

React Router, for example, includes the query params as a string with every Route component via the `location` object.
They also provide url segments, which may also be handy, but for now, may be outside the scope of this RFC.
