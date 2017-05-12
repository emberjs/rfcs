- Start Date: 2016-02-01
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)


# Summary

Improve the route definition process by migrating away from using `this.route(...)` to using
`r.route(...)` where `r` is the argument provided to the function callback at each level.
Using `this.route(...)` will continue to work for backwards compatibility.


# Motivation

The primary purpose of this change is to eliminate confusion about what `this` actually is
when defining routes. Using a callback argument helps to clarify and enforce that it's changing at
each level, as opposed to `this` which is seemingly the same throughout the hierarchy.

A secondary purpose of this change is to enable the use of ES6 arrow functions, which are viewed by many
as more elegant.


# Detailed design

The idea for this came from the way that [ember-test-helpers](https://github.com/switchfly/ember-test-helpers)
passes it's `assert` helper into each test as the function argument.

When declaring an application's routes instead of using `this.route(...)` you could use
`r.route(...)`. Given this example:

```js
Router.map(function() {
  this.route('search', function() {
    this.route('index');
    this.route('advanced');
  });
});
```

It would instead be written as:

```js
Router.map(r => {
  r.route('search', r => {
    r.route('index');
    r.route('advanced');
  });
});
```

*Note: Using `r` as the variable could be documented more verbosely as `router` or anything else.*

I believe this change only involves touching [Ember](https://github.com/emberjs/ember.js).
On first look, it does not appear that [tildeio/route-recognizer](https://github.com/tildeio/route-recognizer)
or [tildeio/router.js](https://github.com/tildeio/router.js) are affected.

These proposed changes are backwards compatible with existing API for defining routes. Ideally
using `this.route` would be deprecated in a future version and removed in Ember 3.0. But I think
that is outside the scope of this RFC.


# Drawbacks

It's possible this could introduce some confusion about the support of `this` vs. the argument,
as well as confusion about using arrow functions with older versions of Ember/Router.
Mixing arrow functions and using `this` can never be supported, so it must be carefully
documented and messaged to the user.

# Alternatives

Since the primary goal of this is to eliminate confusion and prevent new users from tripping up,
it's possible a linting solution such as ember-suave could be applied to ensure the arrow function
is not used for route definitions. However, this would require that the user is actually using the linter.


# Unresolved questions

There are no unresolved questions that I am aware of.
