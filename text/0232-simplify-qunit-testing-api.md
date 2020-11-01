---
Start Date: 2017-06-13
RFC PR: emberjs/rfcs#232
Ember Issue: (leave this empty)

---

# Summary

In order to embrace newer features being added by QUnit (our chosen default
testing framework), we need to reduce the brittle coupling between `ember-qunit`
and QUnit itself. 

This RFC proposes a new testing syntax, that will expose QUnit API directly while
also making tests much easier to understand. 

# Motivation

QUnit feature development has been accelerating since the ramp up to QUnit 2.0.
A number of new features have been added that make testing our applications
much easier, but the current structure of `ember-qunit` impedes our ability 
to take advantage of some of these features.

Developers are often confused by our `moduleFor*` APIs, questions like these
are very common: 

* What "magic" is `ember-qunit` doing? 
* Where are the lines between QUnit and ember-qunit? 
* How can I use QUnit for plain JS objects? 

The way that `ember-qunit` wraps QUnit functionality makes the division
of responsiblity much harder to understand, and leads folks to believe that there
is much more going on in `ember-qunit` than there is. It should be much clearer
what `ember-qunit` is responsible for and what we rely on QUnit for.

This RFC also aims to remove a number of custom testing only APIs that exist today
(largely because the container/registry system was completely private when the
current tools were authored). Instead of things like `this.subject`, `this.register`,
`this.inject`, or `this.lookup` we can rely on the standard way of performing these
functions in Ember via the owner API.

When this RFC has been implemented and rolled out, these questions should all be
addressed and our testing system will both: embrace QUnit much more **and** 
be much more framework agnostic, all the while dropping custom testing only APIs
in favor of public APIs that work across tests and app code.

Sounds like a neat trick, huh?

# Detailed design

The primary change being proposed in this RFC is to migrate to using the QUnit
nested module syntax, and update our custom setup/teardown into a more functional
API.

Lets look at a basic example:

```js
// **** before ****

import { moduleForComponent, test } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';

moduleForComponent('x-foo', {
  integration: true
});

test('renders', function(assert) {
  assert.expect(1);

  this.render(hbs`{{pretty-color name="red"}}`);

  assert.equal(this.$('.color-name').text(), 'red');
});

// **** after ****

import { module, test } from 'qunit';
import { setupRenderingTest } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';

module('x-foo', function(hooks) {
  setupRenderingTest(hooks);

  test('renders', async function(assert) {
    assert.expect(1);

    await this.render(hbs`{{pretty-color name="red"}}`);

    assert.equal(this.$('.color-name').text(), 'red');
  });
});
```

As you can see, this proposal leverages QUnit's nested module API in a way that
makes it much clearer what is going on. It is quite obvious what QUnit is doing
(acting like a general testing framework) and what `ember-qunit` is doing 
(setting up rendering functionality).

This API was heavily influenced by the work that 
[Tobias Bieniek](https://github.com/Turbo87) did in 
[emberjs/ember-mocha#84](https://github.com/emberjs/ember-mocha/pull/84).

## QUnit Nested Modules API

Even though it is not a proposal of this RFC, the QUnit nested module
syntax may seem foreign to some folks so lets briefly review.

With nested modules, a normal 1.x QUnit module setup changes from:

```js
QUnit.module('some description', {
  before() {},
  beforeEach() {},
  afterEach() {},
  after() {}
});

QUnit.test('it blends', function(assert) {
  assert.ok(true, 'of course!');
});
```

Into:

```js
QUnit.module('some description', function(hooks) {

  hooks.before(() => {});
  hooks.beforeEach(() => {});
  hooks.afterEach(() => {});
  hooks.after(() => {});

  QUnit.test('it blends', function(assert) {
    assert.ok(true, 'of course!');
  });
});
```

This makes it much simpler to support multiple `before`, `beforeEach`, `afterEach`,
and `after` callbacks, and it also allows for arbitrary nesting of modules.

You can read more about QUnit nested modules
[here](http://api.qunitjs.com/QUnit/module#nested-module-nested-hooks-). The new APIs
proposed in this RFC expect to be leveraging nested modules. 

## New APIs

The following new methods will be exposed from `ember-qunit`:

```ts
interface QUnitModuleHooks {
  before(callback: Function): void;
  beforeEach(callback: Function): void;
  afterEach(callback: Function): void;
  after(callback: Function): void;
}

declare module 'ember-qunit' {
  // ...snip... 
  export function setupTest(hooks: QUnitModuleHooks): void;
  export function setupRenderingTest(hooks: QUnitModuleHooks): void;
}
```

### `setupTest`

This function will:

* invoke `ember-test-helper`s `setContext` with the tests context
* create an owner object and set it on the test context (e.g. `this.owner`)
* setup `this.set`, `this.setProperties`, `this.get`, and `this.getProperties` to
  the test context
* setup `this.pauseTest` and `this.resumeTest` methods to allow easy pausing/resuming
  of tests

### `setupRenderingTest`

This function will:

* run the `setupTest` implementation
* setup `this.$` method to run jQuery selectors rooted to the testing container 
* setup a getter for `this.element` which returns the DOM element representing
  the element that was rendered via `this.render`
* setup Ember's renderer and create a `this.render` method which accepts a 
  compiled template to render and returns a promise which resolves once rendering
  is completed
* setup `this.clearRender` method which clears any previously rendered DOM (
  also used during cleanup)

When invoked, `this.render` will render the provided template and return a
promise that resolves when rendering is completed.

## Changes from Current System

Here is a brief list of the more important but possibly understated changes
being proposed here:

* the various setup methods no longer need to know the name of the object under test
* `this.subject` is removed in favor of using the standard public API for looking up
  and creating instances (`this.owner.lookup` and `this.owner.factoryFor`)
* `this.inject` is removed in favor of using `this.owner.lookup` directly
* `this.register` is removed in favor of using `this.owner.register` directly
* `this.render` will begin being asynchronous to allow for further iteration in the
  underlying rendering engines ability to speed up render times (by yielding back
  to the browser and not blocking the main thread)
* `this.pauseTest` and `this.resumeTest` are being added
* `this.element` is being introduced as a public API for DOM assertions in a jQuery-less
  environment
* QUnit nested modules are required

These changes generally do not affect our ability to write a codemod to aide in the migration.

## Migration Examples

The migration can likely be largely automated (following the 
[excellent codemod](https://github.com/Turbo87/ember-mocha-codemods) that 
[Tobias Bieniek](https://github.com/turbo87) wrote for a similar `ember-mocha`
the transition), but it is still useful to review concrete scenarios
of tests before and after this RFC is implemented.

### Component / Helper Integration Test

```js
// **** before ****

import { moduleForComponent, test } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';

moduleForComponent('x-foo', {
  integration: true
});

test('renders', function(assert) {
  assert.expect(1);

  this.render(hbs`{{pretty-color name="red"}}`);

  assert.equal(this.$('.color-name').text(), 'red');
});

// **** after ****

import { module, test } from 'qunit';
import { setupRenderingTest } from 'ember-qunit';
import hbs from 'htmlbars-inline-precompile';

module('x-foo', function(hooks) {
  setupRenderingTest(hooks);

  test('renders', async function(assert) {
    assert.expect(1);

    await this.render(hbs`{{pretty-color name="red"}}`);

    assert.equal(this.$('.color-name').text(), 'red');
  });
});
```

### Component Unit Test

```js
// **** before ****

import { moduleForComponent, test } from 'ember-qunit';

moduleForComponent('x-foo', {
  unit: true
});

test('computes properly', function(assert) {
  assert.expect(1);

  let subject = this.subject({
    name: 'something'
  });

  let result = subject.get('someCP');
  assert.equal(result, 'expected value');
});

// **** after ****

import { module, test } from 'qunit';
import { setupTest } from 'ember-qunit';

module('x-foo', function(hooks) {
  setupTest(hooks);

  test('computed properly', function(assert) {
    assert.expect(1);

    let Factory = this.owner.factoryFor('component:x-foo');
    let subject = Factory.create({
      name: 'something'
    });

    let result = subject.get('someCP');
    assert.equal(result, 'expected value');
  });
});
```

### Service/Route/Controller Test

```js
// **** before ****

import { moduleFor, test } from 'ember-qunit';

moduleFor('service:flash', 'Unit | Service | Flash', {
  unit: true
});

test('should allow messages to be queued', function (assert) {
  assert.expect(4);
  
  let subject = this.subject();
  
  subject.show('some message here');
  
  let messages = subject.messages;
  
  assert.deepEqual(messages, [
    'some message here'
  ]);
});

// **** after ****

import { module, test } from 'qunit';
import { setupTest } from 'ember-qunit';

module('Unit | Service | Flash', function(hooks) {
  setupTest(hooks);
  
  test('should allow messages to be queued', function (assert) {
    assert.expect(4);
  
    let subject = this.owner.lookup('service:flash');
  
    subject.show('some message here');
  
    let messages = subject.messages;
  
    assert.deepEqual(messages, [
      'some message here'
    ]);
  });
});

```

## Ecosystem Updates

The blueprints in all official projects (and any provided by popular addons) 
will need to be updated to detect `ember-qunit` version and emit the correct
output.

This includes:

* ember-source
* ember-data
* ember-cli-legacy-blueprints
* others?

This exact process was done for `ember-mocha`'s migration, making this a well
trodden path.

## Update Guides

The guides includes a section for testing, this section needs to be reviewed
and revamped to match the proposal here.

## Deprecate older APIs

Once this RFC is implemented, the older APIs will be deprecated and retained 
for a full LTS cycle (assuming speedy landing, this would mean the older APIs 
would be deprecated around Ember 2.20). After that timeframe, the older APIs
will be removed from `ember-qunit` and `ember-test-helpers` and they will
release with SemVer major version bumps.

Note that while the older `moduleFor` and `moduleForComponent` APIs will be
deprecated, they will still be possible to use since the host application can
pin to a version of `ember-qunit` / `ember-test-helpers` that support its own
usage. This is a large benefit of migrating these testing features away from 
`Ember`'s internals, and into the addon space.

## Relationship to "Grand Testing Unification"

This RFC is a small stepping stone towards the future where all types of tests
share a similar API. The API proposed here is much easier to extend to provide
the functionality that is required for [emberjs/rfcs#119](https://github.com/emberjs/rfcs/pull/119).

# How We Teach This

This change requires updates to the API documentation of `ember-qunit` and the
main Ember guides' testing section. The changes are largely intended to reduce
confusion, making it easier to teach and understand testing in Ember.

# Drawbacks

## Churn

As mentioned in [emberjs/rfcs#229](https://github.com/emberjs/rfcs/pull/229), test
related churn is quite painful and annoying. In order to maintain the general
goodwill of folks, we must ensure that we avoid needless churn. 

This RFC should be implemented in conjunction with 
[emberjs/rfcs#229](https://github.com/emberjs/rfcs/pull/229) so that we can avoid
multiple back to back changes in the blueprints.

## [qunitjs/qunit#977](https://github.com/qunitjs/qunit/issues/977)

Until very recently, the QUnit nested module API was only able to allow a single
callback for each of the hooks per-nesting level. This means that the proposal in
this RFC (which requires the hooks to be setup by `ember-qunit`) would disallow
user-land `beforeEach`/`afterEach` hooks to be setup.

The work around is "simple" (if somewhat annoying), which is to "just nest another
level". The good news is that [Trent Willis](https://github.com/trentwillis) fixed
the underlying problem in [qunitjs/qunit#1188](https://github.com/qunitjs/qunit/pull/1188),
which should be released as 2.3.4 well before this RFC is merged.

# Alternatives

The simplest alternative is to do nothing. This would loose all of the positive
benefits mentioned in this RFC, but should still be considered a possibility...

# Unanswered Questions

## `hooks` argument

A few folks (e.g. [@ebryn](https://github.com/ebryn) and [@stefanpenner](https://github.com/stefanpenner))
have approached me with concerns around the `hooks` argument I have mentioned/used here. The concerns
are generally an initial reaction to the QUnit nested modules API in general and not directly related
to this RFC (other than it highlighting a new feature that they haven't used before).

The main concerns are:

* Teaching folks what `hooks` means is a bit more difficult because it does not represent the "test
  environment", but rather just a way to invoke the callbacks for `before` / `beforeEach` / `after` /
  `afterEach`.
* Passing only `hooks` to the helper functions proposed in the RFC means that if we ever need to thread
  more information through, we either have to use `hooks` as a transport or change our API to add more
  arguments.
* It seems somewhat impossible to communicate across multiple helpers (again without using `hooks`
  as a state/transport mechanism).

I've kicked off a conversation over with the QUnit folks in https://github.com/qunitjs/qunit/issues/1200.
If that PR were merged this proposal would be modified to the following syntax:

```js
// current proposal
module('x-foo', function(hooks) {
  setupRenderingTest(hooks);
  // ....snip....
});

// after qunitjs/qunit#1200
module('x-foo', function(hooks) {
  setupRenderingTest(this);
  // ....snip....
});
```

Another possible solution is to rename the argument (here and in the blueprints) to `module`.
This is more in line with what the QUnit folks view it as: the "module context" that
is being created for that specific `QUnit.module` invocation.
