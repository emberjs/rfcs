---
stage: recommended
start-date: 2024-05-11T00:00:00.000Z
release-date:
release-versions:
  ember-data: 5.3.0
teams:
  - data
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1026'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1035'
  released: 'https://github.com/emberjs/rfcs/pull/1047'
  recommended: 'https://github.com/emberjs/rfcs/pull/1051'
project-link:
suite:
---

# EmberData | Deprecate Store extending EmberObject

## Summary

This RFC deprecates the Store extending from EmberObject. All EmberObject specific
APIs included.

## Motivation

There are two motivations:

First, extending EmberObject is vestigial. The Store makes no use of any EmberObject API,
not even for use with Ember's container or service injection.

Second, in order to support any Ember version, support any non-Ember framework, and support
EmberData running in non-browser environments we want to remove unnecessary coupling to the Ember framework.

## Detailed design

Instead of deprecating every EmberObject method, we will feature flag the Store extending
EmberObject at the module level. This ensures the deprecation only prints once, and that
once resolved the Store will no longer extend thereby making it feasible to utilize the
benefits of not extending EmberObject immediately.

To resolve the deprecation, users will need to confirm they are not using EmberObject APIs
on the Store. Generally speaking, this has been limited to `.extend` e.g.

```ts
const AppStore = Store.extend({});
```

This pattern is now rare in the wild, but where it exists can be safely refactored to

```ts
class AppStore extends Store {}
```

Once confirmed (or in order to confirm) that the Store in an app no longer requires
extending EmberObject, the deprecation config boolean may be used to both remove the
deprecation AND the deprecated code.

```ts
const app = new EmberApp(defaults, {
  emberData: {
    deprecations: {
      DEPRECATE_STORE_EXTENDS_EMBER_OBJECT: false
    }
  }
});
```

An upcoming shift in how EmberData manages configuration would mean that applications
using the new configuration (not yet released) would do the following:

```ts
'use strict';

const EmberApp = require('ember-cli/lib/broccoli/ember-app');

module.exports = async function (defaults) {
  const { setConfig } = await import('@warp-drive/build-config');

  const app = new EmberApp(defaults, {});

  setConfig(app, __dirname, {
    deprecations: {
      DEPRECATE_STORE_EXTENDS_EMBER_OBJECT: false
    }
  });

  return app.toTree();
};
```

## How we teach this

Guides would be added for this deprecation to both the deprecation app and the API docs.

Generally, folks do not tend to treat the Store as an EmberObject or utilize legacy EmberObject
APIs with it, so both the teaching and the migration overhead are low.

## Drawbacks

none

## Alternatives

- deprecate every classic method to help folks find usage
    - not chosen as it's rare *and* setting the deprecation flag to `false` will cause any such locations to be findable via error
- create a new package `@warp-drive/core` or `@warp-drive/store` and have users migrate by swapping import
  locations.
    - not chosen as this is too minimal a change

## Unresolved questions

None
