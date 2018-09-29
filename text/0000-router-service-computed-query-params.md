- Start Date: 2018-09-21
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Query Params as a Computed Property on the RouterService

## Summary

Access to query params is currently restricted to the controller, and subsequently, the corresponding route. 
This results in some limitations with which users may consume query param data. 
By exposing query params as a computed property on the `RouterService`, users will be able to easily access the params from deep down a component tree, removing the need to pass the params down many levels.

## Motivation

Modern SPA concepts have converged on the idea that query params should be easily accessible from the object responsible for handling the route.
Like with the [RouterService](https://github.com/emberjs/rfcs/blob/master/text/0095-router-service.md), 
it is common to have a need to perform routing behavior from deep down a component tree. 

### Examples

Having query params accessible on the router service would allow users to implement:

 - query param aware modals that may hide or show depending on the presence of a query param.
 - fill in form fields from a link.
 - filter / search components could update the query param property.
 - whatever else query params are used for outside of a SPA.

## Detailed design

Add a computed property that splits out `window.location.search` into an object, which could have deeply nested objects and arrays.
The computed property must contain a dependent key on the route path, as the same queury parame may be used differently on different routes.
The computed property must return an object containing computed properties for allowing behavior bound to the value of a query param.

Ensure that setting any deeply nested value in the query params object computed property also updates the URL.

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

Having computed properties available from the `RouterService` would look like this:

```ts
export default class extends Controller {
  @service router;
 
  @readOnly('router.queryParams') query;
 
  @alias('query.articles_category') category;
  @alias('query.page') page = 1;
  @alias('query.filter') filter = 'recent';
 
  @computed('category', 'model')
  get filteredArticles() {
    // do something with category and model as category changes
  }
}
```

## Drawbacks

Anyone relying on behavior of the current way query params are implemented may not be able to _exactly_ have the same behavior. 
This couples the query params to the `RouterService`. 
This may be a good thing, as the current way queryParams are used is somewhat awkward. 

## Alternatives

React Router, for example, includes the query params as a string with every Route component via the `location` object.
They also provide url segments, which may also be handy, but for now, may be outside the scope of this RFC.

The `RouterService` also has a `location` object, which could have the `search` property added to it, 
like the native [location](https://developer.mozilla.org/en-US/docs/Web/API/HTMLHyperlinkElementUtils/search) has.

If only `search` was exposed, it would allow people to continue to use the controller implementation of query params, 
and then define a series of computed properties that depend on `'location.search'` to parse out their query param values.

## Unresolved questions

- What behavior are people using with query params that computed properties defined on the `RouterService` would not allow?
