---
Start Date: 2017-11-05
RFC PR: emberjs/rfcs#268
Ember Issue: (leave this empty)

---

# Summary

The testing story in Ember today is better than it ever has been. It is now
possible to test individual component/template combos, register your own mock
components/services/etc, build complex acceptance tests, and almost anything else
you would like.

Unfortunately, there is a massive disparity between different types of tests.
In acceptance tests, you use well designed global helpers to deal with async
related interactions; whereas in integration and unit tests you are forced to
manually deal with this asynchrony.
[emberjs/rfcs#232](https://github.com/emberjs/rfcs/blob/master/text/0232-simplify-qunit-testing-api.md)
introduced us to QUnit's nested modules API, made integration and unit testing
modular, and greatly simplified the concepts needed to learn how to write unit
and integration tests. The goal of this RFC is to leverage what we have learned
in prior RFCs and apply that knowledge to acceptance testing. Once this RFC has
been implemented all test types in Ember will have a unified cohesive structure.

# Motivation

Usage of rendering tests is becoming more and more common, but these tests
often include manual event delegation (`this.$('.foo').click()` for
example), and assumes most (if not all) interactions are synchronous.  This is
a major issue due to the fact that the vast majority of interactions will
actually be asynchronous. There have been a few recent additions to
`@ember/test-helpers` that have made dealing with asynchrony better (namely
[emberjs/rfcs#232](https://github.com/emberjs/rfcs/blob/master/text/0232-simplify-qunit-testing-api.md))
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
[@ember/test-helpers](https://github.com/emberjs/ember-test-helpers/) library so
that we can iterate faster while supporting multiple Ember versions
independently and easily support multiple testing frameworks build on top of
the primitives in `@ember/test-helpers`. Ultimately, the existing [ember-testing](https://github.com/emberjs/ember.js/tree/master/packages/ember-testing) system
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
import { setupApplicationTest } from 'ember-qunit';
import { visit, fillIn, click } from '@ember/test-helpers';

module('Acceptance | login', function(hooks) {
  setupApplicationTest(hooks);

  test('should add new post', async function(assert) {
    await visit('/posts/new');
    await fillIn('input.title', 'My new post');
    await click('button.submit');

    assert.equal(this.element.querySelectorAll('ul.posts li')[0].textContent, 'My new post');
  });
});
```

As you can see, this proposal unifies on Qunit's nested module syntax following
in emberjs/rfcs#232's footsteps.

## New APIs Proposed

The following new methods will be exposed from `ember-qunit`:

```ts
declare module 'ember-qunit' {
  // ...snip... 
  export function setupApplicationTest(hooks: QUnitModuleHooks): void;
}
```

### DOM Interaction Helpers

New native DOM interaction helpers will be added to both `setupRenderingTest`
and (proposed below) `setupApplicationTest`. The implementation for these
helpers has been iterated on and is quite stable in the
[ember-native-dom-helpers](https://github.com/cibernox/ember-native-dom-helpers)
addon.

The helpers will be migrated to `@ember/test-helpers` and eventually
(once "the dust settles") `ember-native-dom-helpers` will be able to reexport
the versions from `@ember/test-helpers` directly (which means apps that have
already adopted will have very minimal changes to make).

The specific DOM helpers to be added to the `@ember/test-helpers` module are:

```ts
/**
  Clicks on the specified selector.
*/
export function click(selector: string | HTMLElement): Promise<void>;

/**
  Taps on the specified selector.
*/
export function tap(selector: string | HTMLElement): Promise<void>;

/**
  Triggers a keyboad event on the specified selector.
*/
export function triggerKeyEvent(
  selector: string | HTMLElement,
  eventType: 'keydown' | 'keypress' | 'keyup',
  keyCode: string,
  modifiers?: {
    ctrlKey: false,
    altKey: false,
    shiftKey: false,
    metaKey: false
  }
): Promise<void>;

/**
  Triggers an event on the specified selector.
*/
export function triggerEvent(
  selector: string | HTMLElement,
  eventType: string,
  eventOptions: any
): Promise<void>;

/**
  Fill in the specified selector's `value` property with the provided text.
*/
export function fillIn(selector: string | HTMLElement, text: string): Promise<void>;

/**
  Focus the specified selector.
*/
export function focus(selector: string | HTMLElement): Promise<void>;

/**
  Unfocus the specified selector.
*/
export function blur(selector: string | HTMLElement): Promise<void>;

/**
  Returns a promise which resolves when the provided callback returns a truthy value.
*/
export function waitUntil<T>(Function: Promise<T>, { timeout = 1000 }): Promise<T>;

/**
  Returns a promise which resolves when the provided selector (and count) becomes present.
*/
export function waitFor(selector: string, { count?: number, timeout = 1000 }): Promise<HTMLElement | HTMLElement[]>;
```

### `setupApplicationTest`

This function will:

* invoke `ember-test-helper`s `setupContext` with the tests context (which does the following):
  * create an owner object and set it on the test context (e.g. `this.owner`)
  * setup `this.pauseTest` and `this.resumeTest` methods to allow easy pausing/resuming
    of tests
* add routing related helpers
  * setup importable `visit` method to visit the given url
  * setup importable `currentRouteName` method which returns the current route name
  * setup importable `currentURL` method which returns the current URL
* add DOM interaction helpers (heavily influenced by @cibernox's lovely addon [ember-native-dom-helpers](https://github.com/cibernox/ember-native-dom-helpers))
  * setup a getter for `this.element` which returns the DOM element representing
    the applications root element
  * setup importable `click` helper method
  * setup importable `tap` helper method
  * setup importable `triggerKeyEvent` helper method
  * setup importable `triggerEvent` helper method
  * setup importable `fillIn` helper method
  * setup importable `focus` helper method
  * setup importable `blur` helper method
  * setup importable `waitUntil` helper method
  * setup importable `waitFor` helper method

### `setupRenderingTest`

The `setupRenderingTest` function proposed in
[emberjs/rfcs#232](https://github.com/emberjs/rfcs/blob/master/text/0232-simplify-qunit-testing-api.md)
(and implemented in
[ember-qunit](https://github.com/emberjs/ember-qunit)@3.0.0) will be modified to add the same DOM interaction helpers mentioned above:

* setup importable `click` helper method
* setup importable `tap` helper method
* setup importable `triggerKeyEvent` helper method
* setup importable `triggerEvent` helper method
* setup importable `fillIn` helper method
* setup importable `focus` helper method
* setup importable `blur` helper method
* setup importable `waitUntil` helper method
* setup importable `waitFor` helper method

Once implemented, `setupRenderingTest` and `setupApplicationTest` will diverge from each other in very few ways.

## Changes from Current System

Here is a brief list of the more important but possibly understated changes
being proposed here:

* The global test helpers that exist now, will no longer be present (e.g.
  `click`, `visit`, etc) and instead will be available on the test context as
  well as importable helpers.
* `this.owner` will now be present and allow (for the first time 🎉) overriding
  items in the container/registry.
* The new system will leverage the `Ember.Application` /
  `Ember.ApplicationInstance` split so that we can avoid creating an
  `Ember.Application` instance per-test, and instead leverage the same system
  that FastBoot itself uses to avoid running initializers for each acceptance
  test.
* Implicit promise chaining will no longer be present. If your test needs to
  wait for a given promise, it should use `await` (which will wait for the
  system to "settle" in similar semantics to today's `wait()` helper).
* The test helpers that are included by a new default ember-cli app will be no
  longer needed and will be removed from the new application blueprint. This
  includes:
  * `tests/helpers/resolver.js`
  * `tests/helpers/start-app.js`
  * `tests/helpers/destroy-app.js`
  * `tests/helpers/module-for-acceptance.js`

## Examples

### Test Helper

Assuming the following input:

```js
import Ember from 'ember';

export function withFeature(app, featureName) {
  let featuresService = app.__container__.lookup('service:features');
  featuresService.enable(featureName);
}

Ember.Test.registerHelper('withFeature', withFeature);
```

In order for an addon to support both the existing acceptance testing system, and the new system it could replace that helper with the following:

```js
import { registerAsyncHelper } from '@ember/test';

export function enableFeature(owner, featureName) {
  let featuresService = owner.lookup('service:features');
  featuresService.enable(featureName);
}

registerAsyncHelper('withFeature', function(app, featureName) {
  enableFeature(app.__container__, featureName);
});
```

This allows both the prior API (without modification) and the following:

```js
// Option 2:
import { module, test } from 'qunit';
import { setupApplicationTest } from 'ember-qunit';
import { enableFeature } from 'addon-name-here/test-support';

module('asdf', function(hooks) {
  setupApplicationTest(hooks);

  test('awesome test title here', function(assert) {
    enableFeature(this.owner, 'feature-name-here');

    // ...snip...
  });
});
```

### Registering Factory Overrides

Overriding a factory is generally done to allow the test to have more control
over the thing being tested. This is sometimes used to prevent side effects
that are not related to the test (i.e. to prevent network calls), other times
it is used to allow the test to inject some known state to make asserting the
results much easier.

It is currently possible to register custom factories in integration and unit
tests, but not in acceptance tests (without using private API's that is).

As of [emberjs/rfcs#232](https://github.com/emberjs/rfcs/pull/232) the
integration/unit test API for this registration is:

```js
this.owner.register('service:stripe', MockService);
```

This RFC will allow this invocation syntax to work in all test types
(acceptance, integration, and unit).


## Migration

It is important that both the existing acceptance testing system, and the
newly proposed system can co-exist together. This means that new tests can be generated
in the new style while existing tests remain untouched.

However, it is likely that
[ember-qunit-codemod](https://github.com/rwjblue/ember-qunit-codemod) will be
able to accurately rewrite acceptance tests into the new format.

# How We Teach This

This change requires updates to the API documentation of `ember-qunit` and the
main Ember guides' testing section. The changes are largely intended to reduce
confusion, making it easier to teach and understand testing in Ember.

# Drawbacks

* This is a relatively large set of changes that are arguably not needed (things mostly work today).
* One of the major hurdles in upgrading larger applications to newer Ember versions, is updating their tests to follow "new" patterns.  This RFC introduces yet another "new" thing (and proposes to deprecate the old thing), and could therefore be considered "just more churn".

# Alternatives

* Do nothing?
* Make `ember-native-dom-helpers` a default addon (removing the need for DOM interaction helpers proposed here).
