- Start Date: 2015-03-09
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Provide a way to change top level dynamic segments without supplying deeper nested dynamic segments.

# Motivation

For something like a locale in the URL that sits at the top level (`/:locale/products/:product_id/items/:item_id`), there is no easy way to swap it out while preserving the rest of the URL. Here is a [stack overflow question](http://stackoverflow.com/questions/27710301/ember-js-swap-out-a-nested-resource-from-any-route) and an [open issue](https://github.com/emberjs/ember.js/issues/10094).

# Detailed design

This is implemented in an [addon](https://github.com/kellyselden/ember-cli-transition-to-dynamic/). The API there is `this.transitionToDynamic(currentRouteName, { routeWithSegment: { segment: newValue } });`, or `this.transitionToDynamic('item', { product: { id: 1 } });` for the route `/products/:id/items/:id`. This implementation does not cover all cases of `transitionTo`.

Possible API:

```js
this.transitionTo(currentRouteName, {
  segments: {
    routeName: {
      segmentName: newSegment
    }
  }
});
```

For `/:locale/products/:product_id/items/:item_id` it could be:

```js
this.transitionTo(currentRouteName, {
  segments: {
    home: {
      locale: 'en-gb'
    }
  }
});
```

For `/products/:id/items/:id` it could be:

```js
this.transitionTo('item', {
  segments: {
    product: {
      id: 2
    }
  }
});
```

The query params hash could be repurposed as an options hash with segments being another option.

# Drawbacks

Added Router and transitionTo API complexity.

There is already a way to supply dynamic segments through the N number of model arguments. Although, they must be supplied in nested-first order, so you cannot solve this problem.

# Alternatives

Manually supplying the URL using the discouraged API `this.transitionTo('/en-gb/products/1/items/1');`

# Unresolved questions

This could be expanded to cover nested route swaps as mentioned [here](https://github.com/emberjs/ember.js/issues/10094#issuecomment-68450066), but this would add a lot of complexity because you would need a fromRoute and a toRoute.