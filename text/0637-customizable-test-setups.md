---
Start Date: 2020-06-01
Relevant Team(s): Ember-CLI
RFC PR: https://github.com/emberjs/rfcs/pull/637
Tracking: (leave this empty)

---

# Facilitate customization of setupTest* functions

## Summary

Provide a default and convenient way for each project to customize the
`setupApplicationTest`, `setupRenderingTest`, and `setupTest` functions from
[RFC #268](https://github.com/emberjs/rfcs/blob/master/text/0268-acceptance-testing-refactor.md)
and [RFC #232](https://github.com/emberjs/rfcs/blob/master/text/0232-simplify-qunit-testing-api.md).
The app and addon blueprints will be updated to create a file at 
`tests/helpers/index.js` where those functions will be wrapped and exported, 
creating a local place to edit for each type of test setup. Tests generated 
using `ember generate` will import the setup functions from that file.


## Motivation

Projects often need to customize setup for every test. For example, in 
acceptance tests, it is often necessary to override services, mock APIs, etc.
in every test of that type.

A common use-case is `ember-cli-mirage`, which provides a `setupMirage` function
to be called after `setupApplicationTest`. In a project using this, it can be 
necessary to remember to add that call to every test file.

If it is necessary to add new setup to every test (for example, when adding a 
service that must be replaced in test), every test file must be individually 
modified.

For these reasons, many projects create their own `setup*Test` functions in the 
`test/helpers` directory, either wrapping the addon-provided `setup*Test` 
functions or composing with them in each test file.

For example:

```js
  import { module, test } from 'qunit';
  import setupTest from 'ember-qunit';
  
  module('Unit | Service | tomster', function(hooks) {
    setupTest(hooks);

    test('this works', function(...
  });
```

in a generated test where the setup functions have been wrapped in a helper must
manually be changed to:

```js
    import { module, test } from 'qunit';
    import setupTest from 'my-app/tests/helpers';
    
    module('Unit | Service | tomster', function(hooks) {
      setupTest(hooks);
  
      test('this works', function(//...
    });
```

or where the setup functions are to be composed, this must be added:

```js
    import { module, test } from 'qunit';
    import setupTest from 'ember-qunit';
    import setupMyTest from 'my-app/tests/helpers';
    
    module('Unit | Service | tomster', function(hooks) {
      setupTest(hooks);
      setupMyTest(hooks);

      test('this works', function(//...
    });
```

Developers of these projects must remember to change the imports or additionally
call their custom setup in every test file. It is easy to forget to make these 
changes and waste time to debug test failures or weird test side effects.

## Detailed design

When generating tests in an Ember project today, the setup functions in the
generated test are imported from `ember-qunit` or `ember-mocha`. This RFC
proposes adding a file at `test/helpers/index.js` that would look like:

```js
import { setupApplicationTest as upstreamSetupApplicationTest } from 'ember-qunit';
export function setupApplicationTest(hooks, options) {
  upstreamSetupApplicationTest(hooks, options);
  // customize here
}
export { setupTest, setupRenderingTest } from 'ember-qunit';
```

The current imports in newly generated tests:
```js
import { setupApplicationTest } from 'ember-qunit';   
// OR
import { setupRenderingTest } from 'ember-qunit';
// OR
import { setupTest } from 'ember-qunit';
```

will change in the relevant generated tests to: 

```js
import { setupApplicationTest } from 'my-app-name/tests/helpers'; 
// OR
import { setupRenderingTest } from 'my-app-name/tests/helpers';
// OR
import { setupTest } from 'my-app-name/tests/helpers';
```

`ember-qunit` has been used for demonstrative purposes, but the blueprints for 
generated tests using `ember-mocha` are also maintained in `ember-source` 
and `ember-data`. Those blueprints will also be updated to use the local test 
helpers. The `test/helpers/index.js` file will import from the appropriate 
test framework.

We will provide a codemod to update existing tests. @rwjblue has created a 
[proof-of-concept codemod](https://astexplorer.net/#/gist/ba7e5ae104aac099bc5ca60ef874eb74/fc6cd9ad60df7abf17813136d7cdc75b0f313496).

An `eslint-plugin-ember` lint rule will be created to solely allow the imports
of `setup*Test` directly from `ember-qunit`/`ember-mocha` in the 
`test/helpers/index.js` file.

## How we teach this

The Ember.js guides extensively cover testing and use the `setup*Test` functions.
Those guides will need to be updated to reflect the new imports that will be in 
generated tests. Because of this, the guides will need to explain the file at 
`test/helpers/index.js` as a place for customization.

The tutorial also covers testing, using the `setup*Test` functions. It will need
to be updated, as other existing Ember apps will, likely by using the codemod.

This change will not substantially alter how a developer new to Ember would 
learn and use the framework.

## Drawbacks

Developers will need to trace imports through the local file before finding the
the bulk of the setup methods come from `ember-qunit` or `ember-mocha`. 
Projects without a need to customize the behavior of these test setup functions 
may find that slightly inconvenient. 

## Alternatives

We could leave it to each project, as we do now, to create helpers to customize 
behavior as needed. The downsides of this current state would remain as 
discussed in [Motivation](#Motivation).

Another alternative would be to create base classes from which test modules could
extend, similar to how it is done within [`Ember.js`](https://github.com/emberjs/ember.js/blob/master/packages/internal-test-helpers/lib/test-cases/application.js).
This would be a much larger change to how we write tests in Ember projects and 
would have many details to be discussed and hashed out. It may be worth exploring 
but the change proposed in this RFC would still provide value until that idea 
were realized.

### Prior Art

A very similar proposal was discussed in
[a CLI Issue](https://github.com/ember-cli/ember-cli/pull/7657), over two years 
ago.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
