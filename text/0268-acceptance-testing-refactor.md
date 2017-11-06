- Start Date: 2017-11-05
- RFC PR: [emberjs/rfcs#268](https://github.com/emberjs/rfcs/pull/268)
- Ember Issue: (leave this empty)

# Summary

The testing story in Ember today is better than it ever has been. It is now
possible to test individual component/template combos, register your own mock
components/services/etc, build complex acceptance tests, and almost anything else
you would like.

Unfortunately, there is a massive disparity between different types of tests.
In acceptance tests, you use well designed global helpers to deal with async
related interactions; whereas in integration and unit tests you are forced to
manually deal with this asynchrony. The goal of this RFC is to unify the
concepts amongst the various types of test and provide a single common
structure to tests.


# Motivation

Usage of rendering tests is becoming more and more common, but these tests
often still include manual event delegation (`this.$('.foo').click()` for
example), and assumes most (if not all) interactions are synchronous.  This is
a major issue due to the fact that the vast majority of interactions will
actually be asynchronous. There have been a few recent additions to
`ember-test-helpers` that have made dealing with asynchrony better (namely
[emberjs/rfcs#232](https://github.com/emberjs/rfcs/blob/master/text/0232-simplify-qunit-testing-api.md)
but forcing users to manually manage all interaction based async is a recipe
for disaster.

Acceptance tests allow users to handle asynchrony with ease, but they rely on
global helpers that automatically wrap a single global promise which makes
testing of interleaved asynchronous things more difficult. There are a number
of limitations in acceptance tests as compared to integration tests (cannot
mock and/or stub services, cannot look up services to setup test context, etc).

We need a single unified way to teach and understand testing in Ember that
leverages all the things we learned with the original acceptance testing
helpers that were introduced in Ember 1.0.0.  Instead of inventing our own
syntax for dealing with the async (`andThen`) we should use new language
features such as `async` / `await`.

# Detailed design

The goal of this RFC is to introduce new system for acceptance tests that follows in the footsteps of
[emberjs/rfcs#232](https://github.com/emberjs/rfcs/blob/master/text/0232-simplify-qunit-testing-api.md)
and continues to enhance the system created in that RFC to share the same structure and helper system.

This new system for acceptance tests will be implemented in the
[ember-test-helpers](https://github.com/emberjs/ember-test-helpers/) library so
that we can iterate faster while supporting multiple Ember versions
independently and easily support multiple testing frameworks build on top of
the primitives in `ember-test-helpers`. Ultimately, the existing [ember-testing](https://github.com/emberjs/ember.js/tree/master/packages/ember-testing) system
will be deprecated but that deprecation will be added well after the new system has been
released and adopted by the community. 

Lets take a look at a basic example (lifted from [the guides](https://guides.emberjs.com/v2.16.0/testing/acceptance/)):

```js
// **** before ****
import { test } from 'qunit';
import moduleForAcceptance from '../helpers/module-for-acceptance';

moduleForAcceptance('Acceptance | posts');

test('should add new post', function(assert) {
  visit('/posts/new');
  fillIn('input.title', 'My new post');
  click('button.submit');
  andThen(() => assert.equal(find('ul.posts li:first').text(), 'My new post'));
});

// **** after ****
import { module, test } from 'qunit';
import { setupAcceptanceTest } from 'ember-qunit';
import { visit, fillIn, click } from 'ember-test-helpers';

module('Acceptance | login', function(hooks) {
  setupAcceptanceTest(hooks);

  test('should add new post', async function(assert) {
    await visit('/posts/new');
    await fillIn('input.title', 'My new post');
    await click('button.submit');

    assert.equal(find('ul.posts li')[0].textContent, 'My new post');
  });
});
```

As you can see, this proposal is unifies on Qunit's nested module syntax following
in emberjs/rfcs#232's footsteps.

## New APIs Proposed

The following new methods will be exposed from `ember-qunit`:

```ts
declare module 'ember-qunit' {
  // ...snip... 
  export function setupAcceptanceTest(hooks: QUnitModuleHooks): void;
}
```

### `setupAcceptanceTest`

This function will:

* invoke `ember-test-helper`s `setupContext` with the tests context (which does the following):
  * create an owner object and set it on the test context (e.g. `this.owner`)
  * setup `this.set`, `this.setProperties`, `this.get`, and `this.getProperties` to
    the test context
  * setup `this.pauseTest` and `this.resumeTest` methods to allow easy pausing/resuming
    of tests
* add routing related helpers
  * setup `this.visit` method to visit the given url
  * setup getter for `this.currentRouteName` which returns the current route name
  * setup getter for `this.currentURL` which returns the current URL
* add DOM interaction helpers (heavily influenced by @cibernox's lovely addon [ember-native-dom-helpers](https://github.com/cibernox/ember-native-dom-helpers))
  * setup a getter for `this.element` which returns the DOM element representing
    the root element
  * if `jQuery` is present in the application sets up `this.$` method to run
    jQuery selectors rooted to `this.element`
  * setup `this.click` as an async helper to click on the specified selector
  * setup `this.tap` as an async helper to tap on the specified selector
  * setup `this.triggerKeyEvent` as an async helper to trigger a `KeyEvent` on the specified selector
  * setup `this.triggerEvent` as an async helper to trigger an event on the specified selector
  * setup `this.fillIn` as an async helper to enter text on the specified selector
  * setup `this.waitUntil` as an async helper returning a promise which resolves when the provided callback returns a truthy value
  * setup `this.focus` as an async helper which focuses the specified element
  * setup `this.blur` as an async helper which unfocuses the specified element

### `setupRenderingTest`

The `setupRenderingTest` function proposed in
[emberjs/rfcs#232](https://github.com/emberjs/rfcs/blob/master/text/0232-simplify-qunit-testing-api.md)
(and implemented in
[ember-qunit](https://github.com/emberjs/ember-qunit)@3.0.0) will be modified to add the same DOM interaction helpers:

* setup a getter for `this.element` which returns the DOM element representing
  the root element
* if `jQuery` is present in the application sets up `this.$` method to run
  jQuery selectors rooted to `this.element`
* setup `this.click` as an async helper to click on the specified selector
* setup `this.tap` as an async helper to tap on the specified selector
* setup `this.triggerKeyEvent` as an async helper to trigger a `KeyEvent` on the specified selector
* setup `this.triggerEvent` as an async helper to trigger an event on the specified selector
* setup `this.fillIn` as an async helper to enter text on the specified selector
* setup `this.waitUntil` as an async helper returning a promise which resolves when the provided callback returns a truthy value
* setup `this.focus` as an async helper which focuses the specified element
* setup `this.blur` as an async helper which unfocuses the specified element

Once implemented, `setupRenderingTest` and `setupAcceptanceTest` will diverge from each other in very few ways.

### Importable helpers

Since the design of this system relies on both the test helpers being applied
to the test context **and** the usage of `async` / `await`, a few importable
helpers are being introduced to help avoid extra noise (e.g. "rightward shift")
in tests.

This means that users will be able to use either of the following interchangably:

```js
// ***** importable helper *****
import { click } from 'ember-test-helpers';

// ...snip...
test('does something', async function(assert) {
  // ...snip...
  await click('.selector-here');
  // ...snip...
});

// ***** test context helper *****

// ...snip...
test('does something', async function(assert) {
  // ...snip...
  await this.click('.selector-here');
  // ...snip...
});
```

## Changes from Current System

Here is a brief list of the more important but possibly understated changes
being proposed here:

* The global test helpers that exist now, will no longer be present (e.g.
  `click`, `visit`, etc) and instead will be available on the test context as
  well as importable helpers.
* `this.owner` will now be present and allow (for the first time 🎉) overriding
  items in the container/registry.

## Migration

It is quite important that both the existing acceptance testing system, and the
newly proposed system can co-exist together. This means that new tests can be generated
in the new style while existing tests remain untouched.

However, it is quite likely that
[ember-qunit-codemod](https://github.com/rwjblue/ember-qunit-codemod) will be
able to accurately rewrite acceptance tests into the new format.

# How We Teach This

This change requires updates to the API documentation of `ember-qunit` and the
main Ember guides' testing section. The changes are largely intended to reduce
confusion, making it easier to teach and understand testing in Ember.

# Drawbacks

* This is a relatively large set of changes that are arguably not needed (things mostly work today).
* One of the major pains in upgrading larger applications to newer Ember versions, is updating their tests to follow "new" patterns.  This RFC introduces yet another "new" thing (and proposes to deprecate the old thing), and could therefore be considered "just more churn".

# Alternatives

* The primary alternative to this would be to do nothing...

# Unresolved questions

TBD?
