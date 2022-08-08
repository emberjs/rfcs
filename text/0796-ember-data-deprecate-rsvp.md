---
start-date: 2022-02-20T00:00:00.000Z
release-date:
release-versions: 
teams: 
  - data
prs:
  accepted: https://github.com/emberjs/rfcs/pull/796
project-link: 
stage: accepted
---

# Deprecate `RSVP.Promise` for native Promise

## Summary

All methods currently returning an `RSVP.Promise` will return a native Promise. In addition, all documentation will be updated accordingly.

## Motivation

With the removal of IE11 from our build infrastructure and browser requirements, we can now safely remove `RSVP.Promise`. From the [docs](https://github.com/tildeio/rsvp.js/):

> Specifically, it is a tiny implementation of Promises/A+. It works in node and the browser (IE9+, all the popular evergreen ones).

RSVP was Ember's polyfill since an early v1.0.0 version during a time when native Promises didn't exist in any browser.  In addition, Promises have been supported since Node 0.12.

By removing `RSVP.Promise` in favor of native Promises, we can drop an unnecessary dependency for both client side and server side fetching of data.

According to [bundlephobia](https://bundlephobia.com/package/rsvp@4.8.5), this would allow us to remove a significant chunk of dependency weight.

## Detailed design

Two steps will be required to fulfill deprecating `RSVP.Promise`.

First, we will issue a deprecation `DEPRECATE_RSVP_PROMISE` to async callbacks that might be relying on a convenient feature of `RSVP.Promise`.  Namely, an `RSVP.Promise` may still be hanging by the time the underlying model instance or store has been destroyed.  This will help users surface instances where their test suite or code is dealing with dangling promises.  After the removal of this deprecation, this will throw an Error.

- `id: ember-data:rsvp-unresolved-async`

Second, we will also utilize the deprecation to replace all instances of `RSVP.Promise` with native Promises.

The final implementation will look something like:

```js
  ajax(options) {
    if (DEPRECATE_RSVP_PROMISE) {
      return new Promise((resolve, reject) => {
        ...
      });
    } else {
      return new RSVPPromise((resolve, reject) => {
        ...
      });
    }
  },
```

## How we teach this

We do not believe this requires any update to the Ember curriculum. API documentation may be needed to remove traces of `RSVP.Promise`.

See the informal [docs](https://github.com/emberjs/data/blob/fa18fd148e9881a860343eabf0ba15b6f048c3ea/packages/private-build-infra/addon/current-deprecations.ts) on how to configure your compatibility with ember-data deprecations. 

## Drawbacks

For users relying on `RSVP.onerror`, they will have to either refactor their code to set the global Promise to RSVP or configure [onunhandledrejection](https://developer.mozilla.org/en-US/docs/Web/API/WindowEventHandlers/onunhandledrejection) appropriately.

If users continue to use `rsvp` after it is dropped from ember-data, users can add `rsvp` to their package.json explicitly if they were depending on it transitively.

Lastly, RSVP gives meaningful labels so that the promise can be debugged easily. We may need to take this into account with a native Promise wrapper, especially how it interacts with the Ember Inspector.

## Alternatives

Continue resolving async paths with `RSVP.Promise` will allowing users a convenient override to use native Promises.

## Unresolved questions

- What level of an abstraction should we provide over native Promise. Derived state for `isPending`, `isResolved`, and `isRejected` seem like probable derived state we want to expose to users to avoid significant churn in their codebase.
- Consideration of Ember's continued use of RSVP.
- After scanning the codebase, the non native equivalent methods we use from RSVP is `defer` in test files.  This is an implementation detail that doesn't need to be flushed out here.
