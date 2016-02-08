# Grand Testing Unification -- RFC

- Start Date: 2016-02-05
- RFC PR: [emberjs/rfcs#119](https://github.com/emberjs/rfcs/pull/119)
- Ember Issue: N/A

# Summary

The testing story in Ember today is better than it ever has been. It is now possible to test individual component/template combos, register your own mock components/services/etc, build complex acceptance tests, and most anything else you would like.

Unfortunately, there is a massive disparity between different types of tests.  In acceptance tests, you use well designed global helpers to deal with async related interactions; whereas in integration and unit tests you are forced to manually deal with this asynchrony. The goal of this RFC is to unify the concepts amongst the various types of test (acceptance, integration, and unit) and provide a single common structure to tests.

# Motivation

Usage of component integration style tests is becoming more and more common, but these tests still include manual event delegation (`this.$('.foo').click()` for example), and assumes most (if not all) interactions are synchronous.  This is a major issue due to the fact that the vast majority of interactions will actually be asynchronous. There have been a few recent additions to `ember-test-helpers` that have made dealing with asynchrony better, but forcing users to manually manage all async is a recipe for disaster.

Acceptance tests allow users to handle asynchrony with ease, but they rely on global helpers that automatically wrap a single global promise which makes testing of interleaved asynchronous things more difficult. There are a number of limitations in acceptance tests as compared to integration tests (cannot mock and/or stub services, cannot look up services to setup test context, etc).

We need a single unified way to teach and understand testing in Ember that leverages all the things we learned with the original acceptance testing helpers that were introduced in Ember 1.0.0 and improved in 1.2.0.  Instead of inventing our own syntax for dealing with the async (`andThen`) we should use new language features such as `async` / `await`.

# Detailed design

The goal of this RFC is to introduce new constructs, one for each type of test, that all share the same basic structure and helper system. This new system will be implemented in the `ember-test-helpers` library so that it can iterate faster while supporting multiple Ember versions independently.

This proposal will implement a few core concepts to address these issues:

- Usage of `async`/`await` semantics
- Using the same helpers in all types of tests (where applicable)
- New syntax for each test type

### Terminology

Before going too far into this proposal, we should generally agree on a common definition for various terminology used throughout the RFC.

#### Acceptance Tests

Acceptance tests are tests that boot a full application. Acceptance tests are intended to emulate user level interactions like "logging in" and "clicking around" the application to navigate.  They should generally treat the application as a "black box" (though under some circumstances it is acceptable to mock/stub network related services and whatnot). They are the only type of test that have access to a fully booted router and have initializers/instance-initializers ran.

Taking advantage of the container/registry reform that was implemented in Ember 1.12, initializers will run once when the test suite first starts and instance-initializers will run once for each acceptance test. For Ember versions prior to 1.12, we will continue to run initializers for each acceptance test.

#### Integration Tests

Integration tests allow testing smaller sections of the application, such as a given component hierarchy or the interaction between two services.  They are generally testing multiple parts of the system such as: a component and its template; a helper and a service; or even two services interacting with each other. They do not boot an application which means they do not have access to a functioning router and they do not run any of the application initializers.

Integration tests have access to a few custom helpers that provide the ability to render specific template snippets that can be interacted with and tested.

#### Unit Tests

Unit tests are very specifically targeted at creating a single instance of a given factory, and interacting with its methods/actions/etc.  They do not boot an application, have access to the router, or have access to other collaborators within the system without manually specifying each of them. Their purpose is to allow testing cyclomatically complex functions in a concrete way.

### Async / Await

`async`/`await` improves test readability by making it clear that a given helper is async, and avoids the "magical" auto-wrapping of certain global helper functions that exists in Ember's built-in testing helpers.

Consider this snippet using existing acceptance test helpers:

```js
fillIn('.display-title', 'Hello!');
click('.update-display-title');
andThen(() => {
  assert.equal(Ember.$('.display-title').text(), 'Hello!');
});
```

There is no indication when reading this snippet that `click` and `fillIn` are async (the reader just has to know the "magical" async test helper names).

I propose the following replacement snippet:

```js
await this.fillIn('.display-title', 'Hello!');
await this.click('.update-display-title');

assert.equal(this.find('.display-title').text(), 'Hello!');
```

The async/await proposal is currently in Stage 3, and I believe that it has been stable enough to rely on by default.

For more details on `async` / `await` see:

- https://tc39.github.io/ecmascript-asyncawait/
- https://jakearchibald.com/2014/es7-async-functions/

### Shared Test Helpers

By sharing the same test helpers amongst all types of tests (unit, integration, and acceptance) we will have a single consistent mechanism across the board.

#### Always Available

These helpers will be available from all test contexts:

- `pauseTest` - Helper used to pause the test runner and allow interaction.
- `owner` - Property used to return the public owner instance in use by the test.
- `registerWaiter` - Used to instruct the system about unknown asynchrony that should be "waited" for throughout the system (i.e. after an `await this.click(..)` we should wait until AJAX requests are completed).
- `registerHelper` - Used to register a helper with the test context.
- `registerHelpers` - Used to register multiple helpers with the test context.

#### DOM Helpers

These helpers will be available in all testing contexts where DOM helpers are applicable (acceptance and component integration tests). All DOM related helpers will use normal DOM API's, and will avoid using jQuery unless required.

- `click` - Async helper to click on the specified selector.
- `triggerKeyEvent` - Async helper to trigger a `KeyEvent` on the specified selector.
- `triggerEvent` - Async helper to trigger an event on the specified selector.
- `fillIn` - Async helper to enter text on the specified selector.
- `find` - Sync helper to return an element for a given selector (via `document.querySelector`).
- `findAll` - Sync helper to return a list of elements matching the given selector (via `document.querySelectorAll`).
- `element` - Property to access the raw element at the root of the DOM for the given test.

#### Rendering/Integration Helpers

These helpers will be available in testing contexts that allow rendering individual template snippets (integration test):

- `render`  - Async helper to render a given handlebars template.
- `clearRender` - Async helper to clear a previously rendered template.
- `on` - Helper used to register actions on the main test context, for usage within the rendered template.
- `set` Helper used to set properties on the testing context, for usage from within the rendered template.
- `setProperties`  - Helper to allow easier setting of multiple properties (using `set` above).
- `get` - Helper used to get properties from the testing context.
- `getProperties` - Helper used to get multiple properties from the testing context.

#### Routing Helpers

These helpers will be available in testing contexts where routing is being used (acceptance):

- `visit` - Async helper to visit a given url.
- `currentRouteName` - Property containing the current route name.
- `currentURL` - Property containing the current URL.

### New Syntax by Type

In order to unify the way we think about testing, and as an effort to provide backwards compatibility this proposal intends to add new syntax for each test type.

`ember-qunit` would expose the following:

- `moduleForAcceptance`
- `moduleForIntegration`
- `moduleForUnit`

`ember-mocha` would expose the following:

- `describeAcceptance`
- `describeIntegration`
- `describeUnit`

### Common Testing Issues

I propose that we address the following common issues that plague today's testing system:

- Overriding a factory (service for example).
- Registering custom test helpers (async and sync).
- Registering custom waiters to handle foreign async.
- Accessing singleton objects.
- Sharing common setup concerns.

#### Overriding a Factory

Overriding a factory is generally done to allow the test to have more control over the thing being tested. This is sometimes used to prevent side effects that are not related to the test (i.e. to prevent network calls), other times it is used to allow the test to inject some known state to make asserting the results much easier.

It is currently possible to register custom factories in integration and unit tests, but not in acceptance tests (without using private API's that is).

The current integration/unit test API for this registration is:

```js
this.register('service:stripe', MockService);
```

I proposes that we use the public Owner API added in 2.1 and 2.3 to allow this invocation syntax that will work in all test types (acceptance, integration, and unit):

```js
this.owner.register('service:stripe', MockService);
```

#### Registering Custom Test Helpers

It is quite common to want to build a set of shared test helpers to use throughout your test suite.  Giving common interactions a nice name (such as  `loginAsAdmin` or `selectCategory` ) makes writing test boilerplate much simpler, because you do not have to include all the various shared steps in each test.

In acceptance tests, we have two main API's for this today:

- `Ember.Test.registerAsyncHelper`
- `Ember.Test.registerHelper`

These are run eagerly when your test suite is evaluated after which all acceptance tests have access to a global method with the provided helper name.  There are at least a few issues with this system:

- The helpers receive the `Ember.Application` instance as the first argument.  This prevents us from utilizing the new `ApplicationInstance` instances, which would allow our acceptance testing to be much faster.
- Having these helpers automatically augment the testing harness is convenient but often makes tracking down where a given helper is actually defined quite difficult.
- It is also not obvious where you put this global side-effect registration code in a standard ember-cli application.
- This does not work at all for unit or integration tests.

In order to avoid these issues, I propose that test helpers should each be in their own module (`tests/helpers/*.js`) that is imported and registered on the test context:

```js
import moduleForAcceptance from '../helpers/module-for-acceptance';
import { test } from 'ember-qunit';
import selectCategory from '../helpers/select-category';
import loginAsAdmin from '../helpers/login-as-admin';

moduleForAcceptance('Can see things', {
  beforeEach() {
    this.registerHelpers({ loginAsAdmin, selectCategory});
  }
});

test('stuff happens', function(assert) {
  this.loginAsAdmin();
  this.selectCategory('Islay');
});
```

To author these custom helpers, you would do the following:

```js
// tests/helpers/login-as-admin';
import { testHelper } from 'ember-qunit';

export default testHelper(async function() {
  await this.click('.login-now');
  await this.fillIn('.username', 'rwjblue');
  await this.fillIn('.password', 'moar beerz plz');
  await this.click('.submit');
});
```

- Due to the usage of `async` / `await` no special casing of sync vs async helpers is needed.
- The helper shares a context with the test, so it can access the other helpers or properties as needed.  This is somewhat dependent on testing framework (some pass arguments for context while others like QUnit assume an implicit `this` context is provided by the framework).

#### Registering Custom Waiters

When the acceptance test helpers were initially released the system waited on three main things:

- run loop queues to be flushed
- all route async hooks to be completed (`beforeModel`, `model`, `afterModel`, etc)
- pending AJAX requests

It soon became clear that we needed to expose a way for users to influence what is waited on after each async interaction (i.e. `click`). Thus the `Ember.Test.registerWaiters` and `Ember.Test.unregisterWaiters` API's were added.  Unfortunately, these API's were global and users were forced through annoying hoops to use them in a modules world.

With this proposal, I suggest replacing these API's with the `registerWaiters` helper which can be used in a conceptually similar way, but doesn't require global access or "importing for side effects".  Consider the following example:

```js
import { testWaiter } from 'ember-test-helpers';
import { hasPendingTransactions } from 'app-name/services/transactions';

export default testWaiter(function() {
  return hasPendingTransactions();
});
```

After helpers are invoked, the system continues to "wait" until `hasPendingTransactions` returns `true`.

This waiter would be registered in your test as follows:

```js
import moduleForAcceptance from '../helpers/module-for-acceptance';
import { test } from 'ember-qunit';
import transactionWaiter from '../waiters/pending-transactions';


moduleForAcceptance('Can see things', {
  beforeEach() {
    this.registerWaiter(transactionWaiter);
  }
});

test('stuff happens', async function(assert) {
  await this.click('.foo');
  // all pending transactions are completed here...
});

```

#### Accessing Singletons

Accessing singleton objects, such as services, in tests allows easier ability to seed testing data (i.e. through the store), confirm service side effects were made, utilize the service to validate assertions (i.e. translation service), and any number of other use cases.

This is possible today in integration and unit tests like so:

```js
test('foo', function(assert) {
  this.inject.service('store');
  this.store.push(....);
});
```

This was added because at the time, there was no public API to allow consumers to lookup instances from an applications internal container.  However, as of Ember 2.3 we do have a public API for this, and usage of `this.inject.service` is fairly confusing in light of `Ember.inject.service` (which is a different but similar API used by applications).

The above example would be rewritten as:

```js
test('foo', function(assert) {
  this.store = this.owner.lookup('service:store');
  this.store.push(....);
});
```

#### Sharing Common Setup

There are many times when a certain amount of setup for a given type of test is always needed.  It would be unfortunate to force every test to have a bunch of boilerplate (or to break many tests at once by adding an additional common requirement).

To facilitate this, I propose that each application has its own main test module helper for each type of test.  The `tests/helpers/moduleForAcceptance.js` helper would look like:

```js
import { moduleForAcceptance } from 'ember-qunit';
import fooHelper from './foo';

export default function(name, _options) {
  let options = Object.assign({
    helperSetup() {
      this.registerHelper('foo', fooHelper);
    }
  }, _options);

  moduleForAcceptance(name, options);
}
```

This file would not be generated in new ember-cli applications (a default module will be provided for you by `ember-cli-qunit` or  `ember-cli-mocha`), but all newly generated tests will use `../helpers/module-for-acceptance`, `../helpers/module-for-integration`, and `../helpers/module-for-unit` to easily allow customization without modifying every test file.

### Example of Proposed Syntax - QUnit

#### Acceptance Tests

```js
import { moduleForAcceptance } from '../helpers/module-for-acceptance';
import { test } from 'ember-qunit';

moduleForAcceptance('Can see things');

test('clicking on a thing redirects to other-thing', async function(assert) {
  await this.visit('/things');

  let list = this.find('.thing-item');
  assert.equal(list.length, 5);

  await this.click('.thing-item[data-id=1]');

  assert.equal(this.currentRouteName, 'other-thing');
});
```

#### Integration Tests

```js
import { moduleForIntegration } from '../helpers/module-for-integration';
import { test } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';

moduleForIntegration('post-display');

test('expands when clicked', async function(assert) {
  await this.render(hbs`{{post-display}}`);

  assert.equal(this.find('.post-created-at').length, 0, 'precond - no detaisl present');

  await this.click('.post-item');

  assert.equal(this.find('.post-created-at').textContent, '2016-01-05');
});
```

# Migration Plan

Since this is a fairly large departure from our current testing solution, a dedicated migration plan section seems appropriate.

As discussed in the Detailed Design section, I am proposing to introduce new API's that do not currently exist in `ember-test-helpers`, `ember-qunit`, or `ember-mocha`. Once this is done, and the API's have been tested in a number of applications, we will update  the default Ember CLI blueprints to use the new structure and rewrite the guides.emberjs.com testing section. This will allow newly generated tests to use the new API's without immediately breaking existing test suites.

Based on feedback received from the community on the usability of the new structure proposed here, we will deprecate usage of existing Ember API's:

* `Ember.Test.registerHelper`
* `Ember.Test.registerAsyncHelper`
* `Ember.Application#setupForTesting`
* `Ember.Test.unregisterHelper`
* `Ember.Test.onInjectHelpers`
* `Ember.Application#injectTestHelpers`
* `Ember.Application#removeTestHelpers`
* `Ember.Test.registerWaiters`
* `Ember.Test.unregisterWaiters`
* `Ember.Test.*`
* existing `ember-qunit` / `ember-mocha` API's.

It is possible that we can automate some of the migration process, we should continue to investigate migration automation as we progress with implementation.

#### Example Migration of Component Integration Test

This is [an existing test](https://github.com/aptible/dashboard.aptible.com/blob/09c76ef827704c065140fe890ef1a4d24c42be36/tests/integration/components/primitive-select-test.js) for a very simple `{{object-select}}` component:

```js
import Ember from 'ember';
import { moduleForComponent } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';

moduleForComponent('primitive-select', {
  integration: true
});

test('basic attributes are set', function(assert) {
  this.render(hbs('{{primitive-select name="foo"}}'));

  let select = this.$('select');

  assert.equal(select.prop('name'), 'foo', 'provided name was used');
  assert.ok(select.hasClass('form-control'));
});

test('if prompt was supplied it is displayed', function(assert) {
  this.render(hbs('{{primitive-select prompt="Foo"}}'));

  let prompt = this.$('option:contains("Foo")');

  assert.equal(prompt.length, 1, 'prompt was found');
  assert.ok(prompt.prop('disabled'), 'prompt is disabled');
});

test('if no prompt was supplied it is not displayed', function(assert) {
  this.render(hbs('{{primitive-select}}'));

  assert.equal(this.$('option').length, 0, 'prompt was not found');
});

test('provided items are displayed as options', function(assert) {
  this.set('listing', Ember.A(['one', 'two', '3', '4']));

  this.render(hbs('{{primitive-select items=listing}}'));

  let options = this.$('option');
  assert.equal(options.length, 4);
  assert.equal(options.text().trim(), 'onetwo34');
});

test('provided selected item is selected', function(assert) {
  this.set('listing', Ember.A(['one', 'two', '3', '4']));
  this.set('selectedItem', 'two');

  this.render(hbs('{{primitive-select selected=selectedItem items=listing}}'));

  let selectedOption = this.$('option:selected');
  assert.equal(selectedOption.length, 1, 'selected option found');
  assert.equal(selectedOption.text().trim(), 'two', 'correct option is selected');
});

test('when changed fires update action with new value', function(assert) {
  assert.expect(1);

  this.on('checkValue', function(newValue) {
    assert.equal(newValue, 'two');
  });

  this.set('listing', Ember.A(['one', 'two', '3', '4']));

  this.render(hbs('{{primitive-select update=(action "checkValue") items=listing}}'));

  let select = this.$('select');
  Ember.run(function() {
    select.val('two');
    select.trigger('change');
  });
});
```

After updating to this proposal, those tests would look like:

```js
import Ember from 'ember';
import moduleForIntegration from '../helpers/module-for-integration';
import { test } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';

moduleForIntegration('primitive-select');

test('basic attributes are set', async function(assert) {
  await this.render(hbs`{{primitive-select name="foo"}}`);

  let select = this.find('select');

  assert.equal(select.name, 'foo', 'provided name was used');
  assert.ok(select.classList.contains('form-control'));
});

test('if prompt was supplied it is displayed', async function(assert) {
  await this.render(hbs`{{primitive-select prompt="Foo"}}`);

  let prompts = this.findAll('option:contains("Foo")');
  assert.equal(prompts.length, 1, 'prompt was found');

  let prompt = prompts.item(0);
  assert.ok(prompt.disabled, 'prompt is disabled');
});

test('if no prompt was supplied it is not displayed', async function(assert) {
  await this.render(hbs`{{primitive-select}}`);

  assert.equal(this.find('option').length, 0, 'prompt was not found');
});

test('provided items are displayed as options', async function(assert) {
  this.set('listing', Ember.A(['one', 'two', '3', '4']));

  await this.render(hbs`{{primitive-select items=listing}}`);

  let options = this.find('option');
  assert.equal(options.length, 4);
  assert.equal(options.textContent.trim(), 'onetwo34');
});

test('provided selected item is selected', async function(assert) {
  this.set('listing', Ember.A(['one', 'two', '3', '4']));
  this.set('selectedItem', 'two');

  await this.render(hbs`{{primitive-select selected=selectedItem items=listing}}`);

  let [selectedOption, ...rest] = this.findAll('option:selected');
  assert.equal(rest.length, 0, 'only one selected option found');

  assert.equal(selectedOption.textContent.trim(), 'two', 'correct option is selected');
});

test('when changed fires update action with new value', async function(assert) {
  assert.expect(1);

  this.on('checkValue', function(newValue) {
    assert.equal(newValue, 'two');
  });

  this.set('listing', Ember.A(['one', 'two', '3', '4']));

  await this.render(hbs`{{primitive-select update=(action "checkValue") items=listing}}`);

  await this.fillIn('select', 'two');
});
```

# Pedagogy

The main changes that this RFC makes to pedagogy is to massively simplify the set of testing special cases that consumers have to manage mentally while developing an application. The divide between acceptance tests and unit/integration tests largely vanishes, and we begin leveraging language features where we previously implemented a custom DSL.

It is extremely important that as these changes land in the various libararies, we ensure the testing guides listed on guides.emberjs.com are kept up to date. A rewrite of large sections of the existing guide are likely needed, but this rewrite should result in a massive improvement due to the conceptual overhead that is being removed.

# Drawbacks

- This is a relatively large set of changes that are arguably not needed (things mostly work today).
- To debug  `async` / `await` we would increasingly rely on sourcemap accuracy. This might negatively impact debuggability.
- One of the major pains in upgrading larger applications to newer Ember versions, is updating their tests to follow "new" patterns.  This RFC introduces yet another "new" thing (and proposes to deprecate the old thing), and could therefore be considered "just more churn".

# Alternatives

The primary alternative to this would be to do nothing...

# Unresolved questions

- Should we prefer `getOwner(this)` to `this.owner`?  I believe that accessing the owner in testing is much more common (for mocking/stubbing, looking up singleton objects, etc), so we should use `this.owner`.
- Should `this.find` return a jQuery wrapped element? I would prefer to stick with the main DOM API's here, so that we have a chance to share tests between the Fastboot and Browser.
- What "oldest" Ember version should be supported?  I'd prefer if we could make this 1.12 and higher (due to the presence of the container/registry split).
- What browsers do we need to support?  I'd prefer if we matched the Ember 2.x here...
