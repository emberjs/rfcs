---
Start Date: 2020-01-14
RFC PR: (leave this empty)
Ember Issue: (leave this empty)

---

# Summary

Test waiters have been around in Ember in one form or another since version 1.2.0, and provide a way for developers to signal to the testing framework system that async operations are currently active, when to keep waiting, and when those async operations have completed. This allows the active test to wait during the test in a deterministic fashion, and only proceed once the active async is completed.

The current test waiters implementation has a simple but confusing API, and the test waiters themselves lack some key features. This RFC proposes replacing them with a new test waiters system: [ember-test-waiters](https://github.com/rwjblue/ember-test-waiters).

# Motivation

Recently, an updated replacement for the original test waiters API was created. This new library, ember-test-waiters, seeks to provide an easy-to-use API that can be used to interleave unmanaged async behaviors with Ember’s test framework. Using a test waiter can help mark begin and end points for your async operations, allowing the test to deterministically pause during execution.

The new system will provide a few benefits:

1. A new API that removes the existing foot guns (e.g. "Do I return `false` or `true` if I want to continue waiting?")
1. A more robust way to gather debugging information for the test waiter
1. Default test waiters with the ability to author your own, more complex test waiters

# Detailed design

Ember’s test framework has an internal concept of settledness, that is used by all of its internal helpers. Settledness can be defined as **_all known active async operations have completed, and there’s no outstanding work to be done_**. This is codified in the [settled](https://github.com/emberjs/ember-test-helpers/blob/master/API.md#settled) helper.

The settled helper, as noted, wires itself up to known asynchronous behaviors. Those include whether there

- is an active runloop (more on the runloop)
- are any pending timers within the runloop (run.later, run.debounce, run.throttle)
- are any pending test waiters (more on waiters later!)
- are any pending `jQuery.ajax` requests
- are any pending route transitions.

The settled check returns a Promise that is fulfilled when all of the above behaviors return `false`, indicating all async for each behavior is completed. These cover a vast number of async behaviors that are typical in our applications.

An enhancement was added to [`ember-qunit`](https://github.com/emberjs/ember-qunit) that allowed for detecting a lack of settledness at the end of a test. This enhancement, [test isolation validation](https://github.com/emberjs/ember-qunit/blob/master/docs/TEST_ISOLATION_VALIDATION.md), evaluated the settled state once a test was considered done and reported to the user whether there were active async operations pending. This helped developers quickly identify and fix known asynchronous leaks in their tests, allowing for a more deterministic test suite.

During the development of the test isolation validation feature, we discovered that most asynchronous operations used in the settled check provided good debug information that could be provided to the end user, with the exception of the existing test waiters. Those waiters only provided rudimentary information that could be exposed, specifically whether there were any active test waiters pending, but nothing more.

To address this, a new addon was written to experiment on a new test waiter system that would provide a number of things (as noted above):

1. A new API that's _explicit_ and _straightforward_
1. A more robust way to gather debugging information for the test waiter
1. Default test waiters with the ability to author your own, more complex test waiters

This allows developers to utilize `ember-test-waiters` to annotate their asynchronous operations that are not tracked by an `await settled()` check, and for those annotations to provide useful debugging information in the event their async extended past the expected duration of the test.

## Comparison of old waiters system to new

In the old test waiters system, you would do the following:

```js
import { registerWaiter } from '@ember/test';

registerWaiter(function() {
  return myPendingTransactions() === 0;
});
```

While reading the above is straightforward, when writing a test waiter using the old system it's easy to forget what the expected return value is: `true` or `false`. Additionally, it's a bit more cognitive overhead to derive what the intended result of the particular boolean return value is: does returning `true` result in the test waiter waiting or not?

As mentioned before, there's no additional information provided via `registerWaiter`, and capturing stack traces at the call site is currently not implemented. Unmanaged async that 'hangs' can cause your tests to stall and ultimately timeout. Not having stack traces is particularly problematic when trying to identify which of many test waiters has caused this timeout, as it's like looking for a needle in a haystack.

The new system captures an error object when the waiter's `beginAsync` method is called (more on `beginAsync` later), but evaluates the `stack` property lazily, when this value is processed by `@ember/test-helpers`' `getSettledState`. This allows for identifying the offending code more easily.

The new test waiters system looks like this:

```js
import Component from '@ember/component';
import { buildWaiter } from 'ember-test-waiters';

let waiter = buildWaiter('friend-waiter');

export default class Friendz extends Component {
  didInsertElement() {
    let token = waiter.beginAsync();

    someAsyncWork()
      .then(() => {
        //... some work
      })
      .finally(() => {
        waiter.endAsync(token);
      });
  }
}
```

In the above example, a new test waiter is built that is identified via a `name` string passed into the `buildWaiter` function. This allows the waiter to be identifiable, and that name is ultimately used with test isolation validation to help developers narrow down problems in their tests.

## New Test Waiters Design

The new test waiters addon is built using low-level primitives that are complimented with some convenience utilities.

### `Waiter`

At its core, the addon uses an `Waiter` interface defined as follows:

```ts
export type WaiterName = string;
export type Token = unknown;

export interface Waiter {
  name: WaiterName;
  waitUntil(): boolean;
  debugInfo(): TestWaiterDebugInfo[];
}
```

- `name`: The name of the test waiter, which is used to help identify it in test isolation validation output.
- `waitUntil`: Used to determine if the waiter system should still wait for async
  operations to complete. The `waitUntil` method will return `true` to signal completion.
- `debugInfo`: Returns the `debugInfo` for each item tracking async operations in a waiter. The `debugInfo` for each waiter item is ultimately used in `@ember/test-helpers`' `getSettledState` function, which is used for test isolation validation output.

This allows for maximum flexibility when creating your own waiter implementations.

### `TestWaiter`

The `Waiter` interface is built upon to create a more specific interface for a test waiter, `TestWaiter`:

```ts
export interface TestWaiter<T extends object | Primitive | unknown = Token>
  extends Waiter {
  beginAsync(token?: T, label?: string): T;
  endAsync(token: T): void;
  reset(): void;
}
```

- `beginAsync`: Should be used to signal the beginning of an async operation that
  is to be waited for. Invocation of this method should be paired with a subsequent
  `endAsync` call to indicate to the waiter system that the async operation is completed.
- `endAsync`: Should be used to signal the end of an async operation. Invocation of this
  method should be paired with a preceding `beginAsync` call, which would indicate the
  beginning of an async operation.
- `reset`: Resets the waiter state, clearing items tracking async operations in this waiter.

This interface is used for the concrete `TestWaiter` type. This type forms the basis for the addon, and will likely satisfy the majority of use cases.

The most common practice is to import and invoke the `buildWaiter` function to create a new test waiter. The recommendation is to do so at the module level, which allows a single waiter to be created per type (this should likely be enforced via a lint rule added to `eslint-plugin-ember`). A single waiter is then usable across multiple instances.

```ts
function buildWaiter(name: string): TestWaiter;
```

In anything but a production build, this function will return a `TestWaiter` instance. When in production mode, we make this instance _inert_ and essentially no cost to invoke. Since test waiters are intended to be called from application or addon code, but are only required to be _active_ when in tests, this process of making the instance _inert_ is important. Even though code is still invoked, this has a negligible impact on performance.

### Using the `TestWaiter` class

After building a test waiter, most users interact with a limited set of methods within this class, namely `beingAsync` and `endAsync`.

The API used to signal whether an asynchronous operation has begun and ultimately ended is through the **_paired_** calls of `beginAsync` and `endAsync`: begin to denote the start of the asynchronous operation, and end to denote the end. Unique instances of async operations are identified using a `token` returned from `beginAsync`, which is subsequently provided to the `endAsync` call.

To annotate the example provided above:

```js
import Component from '@ember/component';
import { buildWaiter } from 'ember-test-waiters';

// Creates a test waiter with the name 'friend-waiter' that
// is usable by all instances of the `Friendz` component.
let waiter = buildWaiter('friend-waiter');

export default class Friendz extends Component {
  didInsertElement() {
    // Alerts the test waiter system that an async operation has started,
    // storing the resulting unique token to be used to notify the test
    // waiter system that the operation has ended.
    let token = waiter.beginAsync();

    someAsyncWork()
      .then(() => {
        //... some work
      })
      .finally(() => {
        // Notifies the test waiter system that
        // this unique async operation has ended.
        waiter.endAsync(token);
      });
  }
}
```

### `waitForPromise`

The `waitForPromise` utility provides a convenience wrapper around the `TestWaiter` class for use with promises. It ensures the `endAsync` call is invoked in the `finally` of the configured promise.

```js
import Component from '@ember/component';
import { waitForPromise } from 'ember-test-waiters';

export default class MoreFriendz extends Component {
  didInsertElement() {
    waitForPromise(someAsyncWork).then(() => {
      doOtherThings();
    });
  }
}
```

This new test waiters system has been through multiple iterations of refinement, and is in use and integrated with the test isolation validation system.

## Rename of `ember-test-waiters` to `@ember/test-waiters`

We should consider renaming the `ember-test-waiters` repository to `@ember/test-waiters`, which would relocate it to the Ember org and scope it more explicitly.

## Compatibility

The `ember-test-waiters` addon is backwards compatible with the old test waiters system, allowing applications and addons to gradually migrate to using the new system.

Specifically:

- `registerWaiter` continues to work, and is not deprecated
- addons using `ember-test-waiters` work _even if consumed in applications that do not use a new enough `ember-test-helpers` version_ (they won't get additional output via test isolation validation such as test waiter names or stack traces)

The old test waiters system ultimately should be deprecated in its own deprecation RFC.

## Rollout

- Rename `ember-test-waiters` to `@ember/test-waiters`, moving it to the `emberjs` org
- Add `@ember/test-waiters` to the default app and addon blueprints
- Add new optional lint rule to `eslint-plugin-ember` that flags usage of legacy test waiters, recommending the new waiters as a replacement
- Open a new RFC proposing deprecation of legacy test waiters

# How We Teach This

API documentation should be available at `api.emberjs.com`.

## New Testing Section in Ember Guides - "Handling Async"

This new test waiters system should be included in the Ember guide's testing section. Information and examples should be provided to allow users to correctly author asynchronous code that can be correctly managed by the testing system.

Specifically, calling this out in a separate section will allow readers to understand the intent of the test waiters system from a high level, as test waiters apply broadly to all the associated types in the testing section.

There are a few concepts that we should focus on:

1. What patterns of asynchronous code can lead to issues in testing
1. How to utilize test waiters to address those asynchronous patterns
1. How to leverage the built-in test waiters utilities

## New Testing Section in Ember CLI Guides - "Handling Async (for Addon Authors)"

The important requirement that addon authors need to know about is that when using `ember-test-waiters`, that package should be added as a `dependency`.

Specifically for addon authors, we want to encourage the use of test waiters to prevent async leaks in tests due to unmanaged async. A special section directed at addon authors should be added to help them understand the value to their consumers by adding test waiters, or providing test helpers via `addon-test-support` integrated with test waiters.

# Drawbacks

- the new test waiters system ships code to production bundles, though this is quite small

# Alternatives

- the existing test waiters system could be left in place
