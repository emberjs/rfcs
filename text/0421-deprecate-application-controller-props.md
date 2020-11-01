---
Start Date: 2018-19-12
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/421
Tracking: https://github.com/emberjs/rfc-tracking/issues/5

---

# Deprecate Application Controller Router Properties

## Summary 

This RFC proposes the deprecation of `ApplicationController#currentPath` and `ApplicationController#currentRouteName`.

## Motivation

These APIs are no longer needed as the `RouterService` now has `RouterService#currentPath` and `RouterService#currentRouteName`.
These fields are only ever present on the application controller which is a weird special casing that we would like to remove.
Additionally, it's likely that there are very few if any consumers of this API as it is not documented.

## Transition Path

If you are reliant on `ApplicationController#currentPath` and `ApplicationController#currentRouteName` you can get the same functionality from the `RouterService` to migrate, inject the `RouterService` and read the `currentRouteName` or `currentPath` off of it.

Before:

```js
// app/controllers/application.js
import Controller from '@ember/controller';
import fetch from 'fetch';

export default Controller.extend({
  store: service('store'),

  actions: {
    sendPayload() {
      fetch('/endpoint', {
        method: 'POST',
        body: JSON.stringify({
          route: this.currentRouteName
        })
      });
    }
  }
})
```

After:

```js
// app/controllers/application.js
import Controller from '@ember/controller';
import fetch from 'fetch';

export default Controller.extend({
  store: service('store'),
  router: service('router'),

  actions: {
    sendPayload() {
      fetch('/endpoint', {
        method: 'POST',
        body: JSON.stringify({
          route: this.router.currentRouteName
        })
      });
    }
  }
})
```

## How We Teach This

There is likely very few consumers of this functionality and migration path is covered by existing documentation.

## Drawbacks

This may introduce churn that we are not aware of.

## Alternatives

No real alternative other than keep setting the properties.

## Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
