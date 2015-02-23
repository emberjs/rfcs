- Start Date: 2015-02-23
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Optional Segments in Ember routing

## Summary
Introduce optional segments to the routing specification so that unset parameters can map to URLs with missing segments.

## Motivation
### tl;dr
For our application, there are a lot of routes/resources that can be accessed in two modes, default mode and test mode. We'd like the URLs to look like `/test/resource` for test mode and `/resource` for the default mode, which is where our users will spend the majority of their time.

### Real World Stripe Example

As a mechanism for showing why we might like something like this, we'd like you to consider our payments dashboard (which is being re-implemented in Ember). However, we feel the feature could be useful for many generic cases when specificity of top level hierarchies isn't necessary.

It helps to think of the URL as a "selector" of hierarchical data. Because of this, CSS selectors are a good corollary. In our case, we'd like the option to avoid over-specifying the selector. `$('#foo .bar')`is not preferable to `$('.bar')` if `.bar` only exists inside of `#foo` -- however, it's extremely important to specify `#foo` if there are other `.bar` elements present in the DOM.

#### The Stripe [User -> Business -> Mode] Hierarchy

A Stripe user can have (but doesn't have to have) many businesses, and each business can view their data in test mode or live mode (two disjoint data sets). As good stewards of the web, we want users with the correct permissions to be able to directly link their teammates to a resource that is for a specific business in a specific mode.

In the world where a user only has one business and one 'mode' of data, the urls map well to the Ember router system.

`/payments/some_id`

This would retrieve a specific payment from the user's one business in their only "mode." However, if we add the constraint that a user can have live-mode and test-mode datasets, we'd want to reflect that somewhere in the url. This could be a query parameter, but we think that's a little ugly for something so prevalent on our site, and doesn't map well to the data hierarchy (more on this later). Instead we'd like to optionally include the `test/` segment for test mode. This maps most closely to our "Boolean" handling of Optional Segments.

`https://dashboard.stripe.com/payments/some_id`

`https://dashboard.stripe.com/test/payments/some_id` 

This works more nicely in our minds as a way of representing the url, and the hierarchy of the data. You might notice that we could also make this non-optional by always being explicit about the mode:

`https://dashboard.stripe.com/live/payments/some_id`

`https://dashboard.stripe.com/test/payments/some_id`

But we also feel that the default case of 'live' data shouldn't need to be represented in the url. It's "over-specifying" in our opinion since it's not strictly necessary. We'd love to always strive for a 'minimum specific url'.

Additionally, we'd like to handle the business identity. This maps more closely to our 'Dynamic' handling of Optional Segments. Some of our users have many businesses, but the long-tail of users has just one business. We'd like to not trouble users with a single business with an extra url parameter, but still allow deep linking for users with many.

`https://dashboard.stripe.com/payments/some_id`

This would be a valid 'minimum significant url' for a user with a single business in live mode, but in our most complex case (many businesses, many modes):

`https://dashboard.stripe.com/skylight/test/payments/some_id`

This would be the 'minimum significant url' for the 'skylight' business's test data for the payment with an id of `some_id`

However, this is where we run into our first problem with Optional Segment ambiguity, and is the reason that Optional Segments are hard (and deserve discussion). If both the business name and the mode segments are optional, a business with the name `test` would be ambiguous. We outline some of our solutions for handling this situation in our RFC.

#### Hierarchy Semantics, or "Why not Query Params?"

Another solution could be to use query parameters for these cases, but we feel that these two pieces of data are *parents* in the data hierarchy, rather than filters on the total dataset.

Looking at `/skylight/test/payments/some_id` as a tree-like selector implies that you should grab the skylight data node, go into the 'test' data node, go into the payments node list, and then grab the `some_id` leaf node.

Doing the same thing with `/payments/some_id?business=skylight&mode=test` (while also being ugly) implies that you go into the set of all payments data, find all nodes with the `some_id` ID, and then filter them by business name `skylight` and mode `test`. This does not map well onto the representation of the data that we'd like our users to consider (or even on the actual database representation).

The query params solution reminds me of a jQuery `within` plugin. Instead of: `$('body #id .foo')`, it'd be silly to do: `$('.foo').within('body #id')`.

Additionally, persisting query parameters in LinkTo helpers is not something that's currently very easy. For example, when you're on the page `/payments?mode=test`, each of the LinkTo instances would need to link to `/payments/:id?mode=test` - which is odd because we need to pass the mode into every LinkTo and a little bit confusing since we're switching out the "middle" of the url.

#### Phone Numbers (re: Tom Dale's Phone Number -> URL comparison)

867-5309 (Jenny's number) only has 7 digits because she lives in a town where there is only one area code. When I was growing up, all of my friends and I had 7 digit phone numbers. It wasn't until our town grew big enough to require two area codes that we forced the addition of the area code (switched to 10 digits). The area code is an optional phone number segment.

## Alternatives (currently-available solutions in Ember CLI)
Using Ember CLI, currently the only way to do this is to duplicate all routes:

```javascript
Router.map(function() {
  this.resource('testmode', {path: '/test'}, function() {
    this.resource('test-resource1', {path: '/resource1'});
    this.resource('test-resource2', {path: '/resource2'});
    ...
    ...
    ...
    this.resource('test-resource100', {path: '/resource100'});
  });
    
  this.resource('resource1', {path: '/resource1'});
  this.resource('resource2', {path: '/resource2'});
  ...
  ...
  ...
  this.resource('resource100', {path: '/resource100'});
});
```

This forces duplication of each route file due to Ember CLI's conventions, and `controllerName` needs to be overridden to prevent duplication of controllers.

```javascript
// routes/test/test-resource1.js
import Ember from 'ember';
import Resource1Route from './routes/resource1';

export default Resource1Route.extend({
  controllerName: 'resource1';
});
```

Additionally, we'll need to remember to change all usages of `link-to` and other transition helpers (e.g., `Controller#transitionToRoute`) to condition on the `test` parameter, which makes future development of our application (as well as on-boarding new developers) more confusing:

```javascript
{{#if testmode}}
  {{link-to test-resource1 ...}}
{{else}}
  {{link-to resource1 ...}}
{{/if}}
```

Alternatively, several `Ember.Router` methods can be overridden to handle the translation of parameter to segment presence, but this is clearly suboptimal and difficult to maintain.

## Detailed design
We want to introduce the concept of an Optional Segment, which allows a segment in the URL to be optionally present, and would mostly extend from `DynamicSegment`.

Example syntax for defining routes using Optional Segments in Ember CLI:

```javascript
this.resource('mode', {path: '/:test?'}, ...)
```

There are a couple ways to handle routing the resource above:
- **Treat Optional Segments as Booleans**
  - `/test` resolves with params `{ test: true }`
  - `/` resolves with params `{}`
  - If we choose to go with this solution, the syntax could also be something like `/(test)?`
- **Treat Optional Segments as Dynamic Segments when present**
  - `/test` resolves with params `{test: "test"}`
  - `/foo` resolves with params `{test: "foo"}`
  - `/` resolves with params `{}`

### Real world example of using Optional Segments
Based on our proposed solution, here's example syntax for defining routes for our exact use case using Optional Segments in Ember CLI:

```javascript
this.resource('business', {path: '/:business'}, function() {
  // Caveat: business.index can never be reached.
  // e.g. The path `/cookie-shop` will resolve to `business.mode.index` with `params: {}`.
    
  this.resource('mode', {path: '/:mode?}), function() {
    this.resource('payments', function() { ... });
    this.resource('customers', function() { ... });
    this.resource('transfers', function() { ... });
        
    // ...
  });
});
```

Our more advanced use case, where business name is optionally in the route, is uglier, but still possible to accomplish with Optional Segments:

```javascript
this.resource('business', {path: '/:business?/:mode?'}, function() {
  // Here, for something like `/test/customers` we'd get:
  // `{business: 'test', mode: null}`,
  // which we'd then be able to resolve ourselves since both are in scope, and we know
  // that 'test' is not a valid business name.
  this.resource('payments', function() { ... });
  this.resource('customers', function() { ... });
  this.resource('transfers', function() { ... });
        
  // ...
});
```

### Scope of Changes
Most changes will affect `route-recognizer`:
- add new segment type and update `parse`
- `generate` needs special case to prevent extra slash from being written when parameter is not set (see https://github.com/tildeio/route-recognizer/blob/master/lib/route-recognizer.js#L409 for example with `EpsilonSegment`)

## Drawbacks
Adding variable-length segments makes certain combinations of routes ambiguous. For example:

```javascript
// [Example A]
Ember.Router.map(function() {
  this.resource('first', {path: '/:a?'}, function() {
    this.resource('second', {path: '/a'}, function() {
      this.resource('third', {path: '/:a?'}, function() { ... }
    });
  });
});
```

This would result in the following call in `route-recognizer`:
```javascript
router.add([
  {path: '/:a?', handler: ...},
  {path: '/a',   handler: ...},
  {path: '/:a?', handler: ...}
]);
```

In [Example A], while routing the URL `/a/a`, it is ambiguous which optional segment is set. This problem also arises when multiple star segments exist:

```javascript
// [Example B]
Ember.Router.map(function() {
  this.resource('first', {path: '/*a'}, function() {
    this.resource('second', {path: '/*b'}, function() {});
  });
});
```

This would result in the following call in `route-recognizer`:
```javascript
> router.add([
    {path: '/*a', handler: function() {}},
    {path: '/*b', handler: function() {}}
  ])
> router.recognize('/a/b/c/d/e')
{ '0':
   { handler: [Function],
     params: { a: 'a/b/c/d' },
     isDynamic: true },
  '1':
   { handler: [Function],
     params: { b: 'e' },
     isDynamic: true },
  queryParams: {},
  length: 2 }
```

The existing solution is to greedily consume from left to right, which would work in resolving [Example A]:

```javascript
// [Example C]
> router.add([
    {path: '/*a', handler: function() {}},
    {path: '/STATIC',  handler: function() {}},
    {path: '/*a', handler: function() {}}
  ])
> router.recognize('/a/b/c/STATIC/d/e/f')
{ '0':
   { handler: [Function],
     params: { a: 'a/b/c' },
     isDynamic: true },
  '1':
   { handler: [Function],
     params: {},
     isDynamic: false },
  '2':
   { handler: [Function],
     params: { a: 'd/e/f' },
     isDynamic: true },
  queryParams: {},
  length: 3 }
> router.recognize('/a/b/c/STATIC/d/e/f/STATIC/g/h/i')
{ '0':
   { handler: [Function],
     params: { a: 'a/b/c/STATIC/d/e/f' },
     isDynamic: true },
  '1':
   { handler: [Function],
     params: {},
     isDynamic: false },
  '2':
   { handler: [Function],
     params: { a: 'g/h/i' },
     isDynamic: true },
  queryParams: {},
  length: 3 }
```


## Notes
- **Our proposed solution is fully backwards compatible with existing routes.**
- A `StarSegment` matches one or more slash-separated segments, rather than zero or one. For example, `/*foo` matches `/foo/bar` but not `/`, whereas `/:foo?` would match `/` but not `/foo/bar`.
- There are a few alternate proposals that we've considered:
  - Change `StarSegment` so that it matches zero or more instead of one or more segments. We haven't explored this option in depth, and going this route would introduce backwards-incompatibility.
  - (When treating optional segments as dynamic segments only): Another proposal that solves the ambiguity with multiple optional segments in a URL is by applying the option to the entire segment. This would disallow use of the `?` marker anywhere except for the end of a path. For example:
`this.resource('business', {path: '/biz/:id?'})` matches `/biz/foo` but not `/biz`
To match on `/biz` as well, one can use
```javascript
this.route('biz', function() {
  this.resource('business', {path: '/:id?'}, function() {...});
})
```

## Resources
- Our current payments dashboard, built in BackboneJS, which has the live/test URLs we desire: https://dashboard.stripe.com
- Existing ember.js issue with similar problem: [Ember.js#1793](emberjs/ember.js#1793)
- Existing router.js issue with similar problem: [Router.js#102](tildeio/router.js#102)

