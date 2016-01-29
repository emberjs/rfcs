- Start Date: 2016-1-28
- RFC PR:
- Ember Issue:

# Summary

Add `Ember.run.callback()` to support running asynchronous code which tests (specifically `andThen()`) will wait for before considering the app "settled".

# Motivation

Ember's asynchronous test helpers (i.e. `visit()`, `fillIn()`, `andThen()`) provide a robust approach to acceptance testing in the face of asynchronous user interactions and application code. The `andThen()` helper is the backbone of this approach - it waits until the app has "settled" (i.e. async activity has competed) before asserting the desired state.

For many, if not most, ambitious web applications, integration with third party libraries that require asynchronous interaction is a requirement. But unfortunately, there is currently no elegant solution for informing Ember of this asynchronous activity without resorting to lower level APIs like `registerWaiter()`.

For such a common problem, Ember's opinionated approach needs an opinionated answer that matches the elegance of the rest of the test API.

# Detailed design

This RFC adds a single method to the `Ember.run` namespace: `Ember.run.callback()` (better naming suggestions are welcome!).

The method takes a single function as an argument, and returns a wrapper function that can be used in place of the originally supplied one. This wrapper does two things:

1. Ensures the callback is wrapped in an `Ember.run()` call, mostly for convenience.
2. More importantly, it wraps the user-supplied function with a `registerWaiter` method (and the associated state management / cleanup) which resolves once the
callback is invoked.

By having the application register the waiters itself, we can avoid compromising the "purity" of the acceptance tests which would otherwise need to reach into the application's running state.

The return value of of `Ember.run.callback()` would be the wrapped function, but the waiter would be registered immediately. If the callback is no longer needed, the user could cancel the waiter by passing the callback to `Ember.run.cancel()` (matching the other async `Ember.run.*` methods).

The implementation has been spiked out in an addon: [ember-run-callback](https://github.com/davewasmer/ember-run-callback).

# Drawbacks

* This would increase the `Ember.run` API surface and complexity, which (anecdotally) seems to be one of the less understood APIs in Ember.
* There could be some potentially tricky edge cases if a user creates a test-aware callback via this method, but then fails to invoke it. Because invoking `Ember.run.callback()` _immediately_ starts to block the app from settling, if the callback is never invoked, it would hang the tests.

# Alternatives

This could always stay a userland addon rather than be adopted into the core framework.

## Current solutions

There are two main approaches to dealing with this problem currently:

### Manual waiters

```js
click('.start-something-async');
Ember.Test.registerWaiter(function myCustomWaiter() {
  return find('.some-async-result').text().trim() === 'All done!';
});
andThen(function() {
  Ember.Test.unregisterWaiter(myCustomWaiter);
  expect(find('.some-async-result').text().trim()).to.equal('All done!');
});
```

### Direct referencing

```js
click('.start-something-async');
andThen(function() {
  let promise = app.__container__.lookupFactory('controller:foo').get('promise');
  promise.then(function() {
    expect(find('.some-async-result').text().trim()).to.equal('All done!');
  });
});
```

# Unresolved questions

* Are there any performance implications of wrapping the callback with test related code?
* Does this cover enough async use cases to warrant adoption, or are there significant use cases that would be missed?
