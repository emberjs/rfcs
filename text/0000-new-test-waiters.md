- Start Date: 2020-01-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Test waiters have been around in Ember in one form or another since version 1.2.0, and provide a way for developers to signal to the testing framework system that async operations are currently active, when to keep waiting, and when those async operations have completed. This allows the active test to wait during the test in a deterministic fashion, and only proceed once the active async is completed,

The current test waiters implementation has a simple but confusing API, and the test waiters themselves lack some key features. This RFC proposes replacing them with a new test waiters system: [ember-test-waiters](https://github.com/rwjblue/ember-test-waiters).

# Motivation

Recently, an updated replacement for the original test waiters API was created. This new library, ember-test-waiters, seeks to provide an easy-to-use API that can be used to interleave unmanaged async behaviors with Ember’s test framework. Using a test waiter can help mark begin and end points for your async operations, allowing the test to deterministically pause during execution.

The new system will provide a few benefits:

1. A new API that's _explicit_ and _straightforward_
1. A more robust way to gather debugging information for the test waiter
1. Default test waiters with the ability to author your own, more complex test waiters

# Detailed design

Ember’s test framework has an internal concept of settledness, that is used by all of its internal helpers. Settledness can be defined as **_all known active async operations have completed, and there’s no outstanding work to be done_**. This is codified in the [settled](https://github.com/emberjs/ember-test-helpers/blob/master/addon-test-support/@ember/test-helpers/settled.ts#L158) helper.

The settled helper, as noted, wires itself up to known asynchronous behaviors. Those include whether there

- is an active runloop (more on the runloop)
- are any pending timers within the runloop (run.later, run.debounce, run.throttle)
- are any pending test waiters (more on waiters later!)
- are any pending AJAX requests
- are any pending route transitions.

The settled check returns a Promise that is fulfilled when all of the above behaviors return `false`, indicating all async for each behavior is completed. These cover a vast number of async behaviors that are typical in our applications.

An enhancement was added to [`ember-qunit`](https://github.com/emberjs/ember-qunit) that allowed for detecting a lack of settledness at the end of a test. This enhancement, [test isolation validation](https://github.com/emberjs/ember-qunit/blob/master/docs/TEST_ISOLATION_VALIDATION.md), evaluated the settled state once a test was considered done and reported to the user whether there were active async operations pending. This helped developers quickly identify and fix known asynchronous leaks in their tests, allowing for a more deterministic test suite.

During the development of the test isolation validation feature, we discovered that most asynchronous operations used in the settled check provided good debug information that could be provided to the end user, with the exception of the existing test waiters. Those waiters only provided rudimentary information that could be exposed, specifically whether there were any active test waiters pending, but nothing more.

To address this, a new addon was written to experiment on a new test waiter system that would provide a number of things (as noted above):

1. A new API that's _explicit_ and _straightforward_
1. A more robust way to gather debugging information for the test waiter
1. Default test waiters with the ability to author your own, more complex test waiters

This would allow developers to utilize this new test waiters system to annotate their asynchronous operations not tracked by the settled check, and for those annotations to provide useful debugging information in the event their async extended past the expected duration of the test.

## Comparison of old waiters system to new

In the old test waiters system, you would do the following:

```js
import { registerWaiter } from '@ember/test';

registerWaiter(function() {
  return myPendingTransactions() === 0;
});
```

While reading the above is straightforward, when writing a test waiter using the old system it's easy to forget what the expected return value is: `true` or `false`. Additionally, it's a bit more cognitive overhead to derive what the intended result of the particular boolean return value is: does returning `true` result in the test waiter waiting or not?

As mentioned before, there's no additional information provided via `registerWaiter`, and no way to capture stack traces at the call site.

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

The API used to signal whether an asynchronous operation has begun and ultimately ended is through the paired calls of `beginAsync` and `endAsync`: begin to denote the start of the asynchronous operation, and end to denote the end. Unique instances of async operations are identified using a `token` returned from `beginAsync`, which is subsequently provided to the `endAsync` call.

This new test waiters system has been through multiple iterations of refinement, and is in use and integrated with the test isolation validation system. Furthermore, it is backwards compatible with the old test waiters system, allowing applications and addons to gradually migrate to using the new system.

## Deprecating the old test waiters

The old test waiters system will be deprecated in its own deprecation RFC.

# How We Teach This

This new test waiters system should be included in the Ember guide's testing section. Information and examples should be provided to allow users to correctly author asynchronous code that can be correctly managed by the testing system.

# Drawbacks

- added work to integrate the new test waiters system with `ember-mocha`
- the new test waiters system ships code to production bundles, though this is quite small

# Alternatives

- the existing test waiters system could be left in place

# Unresolved questions

- what is the correct order of operations to roll a change like this out?
