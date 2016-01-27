- Start Date: 2016-01-27
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Expose the default starting value of queryParams so they can be reset in an automatic way.

# Motivation

If you want to reset query params for a route now, you have to copy all the default values to reset them in the route's `resetController`, or you have to use internal route data to find the default values. If we expose the defaults, a general case mixin can be created that will reset any arbitrary route's query params.

# Detailed design

Here is the current recommended way to reset query params:

```js
resetController(controller, isExiting) {
  if (isExiting) {
    controller.set('firstParam', 'myDefaultValue');
    controller.set('secondParam', 'myOtherDefaultValue');
  }
}
```

This can't be applied in a general way because it has to know specifics about each route.

Here is a sample from my addon [ember-query-params-reset](https://github.com/kellyselden/ember-query-params-reset/blob/master/addon/mixins/query-params-reset-route.js). This sniffs for the appropriate internals for your ember version:

```js
resetController(controller, isExiting) {
  if (isExiting) {
    this.get('_qp.qps').forEach(qp => {
      let defaultValue;

      if (qp.hasOwnProperty('def')) {
        // < v2.0
        defaultValue = qp.def;
      } else {
        // >= v2.0
        defaultValue = qp.defaultValue;
      }

      controller.set(qp.prop, defaultValue);
    });
  }
}
```

It would be great if there was a public API that this addon could consume so it isn't so brittle.

# Drawbacks

Maintenance of the API.

# Alternatives

Using private internals or manually specifying per route, neither ideal.

# Unresolved questions

What does the API look like? I don't really have a preference.
