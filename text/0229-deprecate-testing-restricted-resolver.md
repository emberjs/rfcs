- Start Date: 2017-06-11
- RFC PR: [emberjs/rfcs#229](https://github.com/emberjs/rfcs/pull/229)
- Ember Issue: (leave this empty)

# Summary

In order to largely reduce the brittleness of tests, this RFC proposes to
remove the concept of artificially restricting the resolver used under
testing.

# Motivation

Disabling the resolver while running tests leads to extremely brittle tests.

It is not possible for collaborators to be added to the object (or one
of its dependencies) under test, without modifying the test itself (even if
exactly the same API is exposed). 

The ability to restrict the resolver is **not** actually a feature of Ember's
container/registry/resolver system, and has posed as significant maintenance
challenge throughout the lifetime of ember-test-helpers.

Removing this system of restriction will make choosing what kind of test to
be used easier, simplify many of the blueprints, and enable much simpler refactoring 
of an applications components/controllers/routes/etc to use collaborating utilties
and services.

# Transition Path

## Deprecate Functionality

Issue a deprecation if `integration: true` is not included in the specified 
options for the APIs listed below. This specifically includes specifying 
`unit: true`, `needs: []`, or specifying none of the "test type options" 
(`unit`, `needs`, or `integration` options) to the following `ember-qunit`
and `ember-mocha` API's:

* `ember-qunit`
  * `moduleFor`
  * `moduleForComponent`
  * `moduleForModel`
* `ember-mocha`
  * `setupTest`
  * `setupComponentTest`
  * `setupModelTest`

### Non Component Test APIs

The migration path for `moduleFor`, `moduleForModel`, `setupTest`, and 
`setupModelTest` is very simple:

```js
// ember-qunit

// before
moduleFor('service:session');

moduleFor('service:session', {
  unit: true
});

moduleFor('service:session', {
  needs: ['type:thing']
});

// after
moduleFor('service:session', {
  integration: true
});
```

```js
// ember-mocha

// before
describe('Session Service', function() {
  setupTest('service:session');
  
  // ...snip...
});

describe('Session Service', function() {
  setupTest('service:session', { unit: true });
  
  // ...snip...
});

describe('Session Service', function() {
  setupTest('service:session', { needs: [] });
  
  // ...snip...
});

// after

describe('Session Service', function() {
  setupTest('service:session', { integration: true });
  
  // ...snip...
});
```

The main change is adding `integration: true` to options (and removing `unit` or `needs`
if present).

### Component Test APIs

Implicitly relying on "unit test mode" has been deprecated for quite some time 
([introduced 2015-04-07](https://github.com/emberjs/ember-test-helpers/pull/38)),
so all consumers of `moduleForComponent` and `setupComponentTest` are specifying
one of the "test type options" (`unit`, `needs`, or `integration`).

This RFC proposes to deprecate completely using `unit` or `needs` options with
`moduleForComponent` and `setupComponentTest`. The vast majority of component tests
should be testing via `moduleForComponent` / `setupComponentTest` with the `integration: true`
option set, but on some rare occaisions it is easier to use the "unit test" style is
desired (e.g. non-rendering test) these tests should be migrated to using `moduleFor` 
/ `setupTest` directly.

```js
// ember-qunit

// before
moduleForComponent('display-page', {
  unit: true
});

moduleForComponent('display-page', {
  needs: ['type:thing']
});

// after

moduleFor('component:display-page', {
  integration: true
});
```

```js
// ember-mocha
describe('DisplayPageComponent', function() {
  setupComponentTest('display-page', { unit: true });
  
  // ...snip...
});

describe('DisplayPageComponent', function() {
  setupComponentTest('display-page', { needs: [] });
  
  // ...snip...
});

// after

describe('DisplayPageComponent', function() {
  setupTest('component:display-page', { integration: true });
  
  // ...snip...
});
```

## Ecosystem Updates

The blueprints in all official projects (and any provided by popular
addons) will need to be updated to avoid triggering a deprecation.

This includes:

* `ember-source`
* `ember-data`
* `ember-cli-legacy-blueprints`
* Others?

# How We Teach This

This RFC would require an audit of the main Ember.js guides to ensure
that all usages of the APIs in question continue to be non-deprecated
valid usages.

# Drawbacks

The primary drawback to this is the churn associated with modifying the 
options passed for each test. This can almost certainly be mitigated by
providing a codemod to enable automated updating.

There are additional changes being entertained that would require changes
for the default testing blueprints, we should ensure that these RFCs do not
conflict or cause undue churn/pain.
