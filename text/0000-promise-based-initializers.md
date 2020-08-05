- Start Date: 2020-01-09
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/572
- Tracking: (leave this empty)

# Promise-based Initializers

## Summary

This RFC proposes the ability to return a promise from an initializer or instance-initializer
and pause application boot until that promise resolves.

## Motivation

**Earlier Async Hooks**

Currently, the earliest place developers can write blocking asynchronous code is
in the ApplicationRoute's `beforeModel()` hook. There are two problems with this:

1. Addons cannot easily add behavior to this hook. To do so, they must reopen the
`Ember.Route` class, and then require that all Routes in the application remember
to call `super`. This approach does not scale well, but will also become harder
as Ember.Route moves away from extending from `CoreObject`, and thus losing the
`.reopen()` API.
1. Semantically, code can make more _sense_ to belong in the application lifecycle,
rather than the routing stack. For example, if your Ember application depends on an
embedded environment (an Electron shell or even an iframe), it may make more sense to
implement blocking code _before_ hitting the routing stack. This can also translate
to technical wins, as initializers run before routing starts, so, for example,
an initializer could determine the starting URL of the application (much like
window.location does today). Yet another example of better codifying developer patterns
is the ability to import or initialize 3rd part libraries with async APIs.

## Detailed Design

The `Application` and `ApplicationInstance` (extending from `Engine` and `EngineInstance`
classes respectively) are responsible for running initializers and instance initializers. Each
implements a method that loops through the loaded initializers and executes the exported function.

Initializer and Instance Initializer files each export an object with an `initialize` property
that references a function. For backwards compatibility, an `async` property should be added, and
should default to `false` if omitted. If true (or truthy), the Application/ApplicationInstance would
expect a promise to be returned from the function and wait for the promise to resolve
before running the next initializer in the queue. The value resolved from the promise would be
ignored. Promises that reject would throw an exception in development and be caught in production
builds.

For example:

```js
// app/initializers/foo.js
export function initialize() {
  return Promise.resolve();
}

export default {
  initialize,
  async: true,
};
```

For maximum backwards compatibility, the boot sequence that calls these initializers could implement
an early check to see if any async initializers are defined and forgo all async handling.

If implemented, future RFCs could:

- change the default value of the `async` property to `true`.
- deprecate `deferReadiness` and `advanceReadiness` entirely.

## How we teach this

1. The guides would need to document the `async` key of the exported object from initializers and instance-initializers.
2. Some advanced logging to see when initializer functions are called may be useful.
3. Some information about how this relates to `deferReadiness` and `advanceReadiness` would be useful.
   It's possible that deprecating these APIs as part of this RFC is appropriate, so there's a clear
   signal that one is preferred over the other.

## Drawbacks

- There could be some performance concerns over introducing promises at boot time, but because this
  is an opt-in feature, they can be avoided.

## **Alternatives**

- The existing `{defer|advance}Readiness` APIs could be introduced for instance-initializers as well.
  Although promise-based async isn't exactly the same as this API, and there could be subtle differences
  between both, it is unlikely that end-users would actually end up caring.
- Deprecate initializers entirely and create new primitives for hooking into the
application boot lifecycle using the `Application` class.

## **Unresolved questions**

- What happens if an initializer is marked as async but doesn't return a promise?
- What happens if a promise is created in an async initializer but not returned?
- Does this affect the algorithm that orders initializers based on `before` and `after` APIs?
- Does this affect the `boot` vs `_bootSync` of Application and ApplicationInstance at all and
  should that be cleaned up _before_ attempting this?
