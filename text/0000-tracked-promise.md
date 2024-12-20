---
stage: accepted
start-date: 2024-12-19T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - data
  - framework
  - learning
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

# Built in tracking utility for promises 

## Summary

This RFC defines a new utility to add by default to the framework, based on all the prior community implementations of promise wrapping, which is byte-wise opt-in in a brand new package (`@ember/reactive`).

## Motivation

Promises are ubiquitous, yet we don't have a _default_ way to use them _reactively_ in our templates.  
Meanwhile, we have several _competing and duplicate_ implementations:
- `TrackedAsyncData` from [ember-async-data][gh-tracked-async-data]
- `PromiseState` from [@warp-drive/ember][gh-ember-data-promise-state]
- `State` from [reactiveweb's remote-data][gh-reactiveweb-state]
- `Request` from [ember-data-resources][gh-ember-data-resource-request]
- `Query` from [ember-await][gh-ember-await-query]

[gh-reactiveweb-state]: https://github.com/universal-ember/reactiveweb/blob/main/reactiveweb/src/remote-data.ts#L13
[gh-tracked-async-data]: https://github.com/tracked-tools/ember-async-data/blob/main/ember-async-data/src/tracked-async-data.ts
[gh-ember-data-promise-state]: https://github.com/emberjs/data/blob/main/packages/ember/src/-private/promise-state.ts
[gh-ember-data-resource-request]: https://github.com/NullVoxPopuli/ember-data-resources/blob/main/ember-data-resources/src/-private/resources/request.ts 
[gh-ember-await-query]: https://github.com/EmberExperts/ember-await/blob/master/addon/utils/query.js#L8

There are probably more, but the point of this RFC is to provide a common utility, that we can both unify all of the implementations and polyfill to earlier versions of ember as well.


## Detailed design

For loadaing UIs' there are concerns that this proposed utility will not be concerned with directly, but also the utility still needs to enable usage of those UIs, so in _How we teach this_, there will be examples of various loading patterns and re-implementations of existing utilities (such as those mentioned in the _Motivation_ section above) using the new tracking utility for promises.


The import:
```js
import { TrackedPromise, trackPromise } from '@ember/reactive';
```

### `trackPromise`

This utility wraps and instruments any promise with reactive state, `TrackedPromise`.  


Sample type declaration
```ts
export function trackPromise<Value>(existingPromise: Promise<Value>): TrackedPromise<Value> {
    /* ... */
}
```

### `TrackedPromise`

This utility is analgous to `new Promise((resolve) => /* ... */)`, but includes the tracking of the underlying promise.
Additionally `TrackedPromise` must be able to receive an existing promise via `new TrackedPromise(existingPromise);`.

Sample type declaration
```ts
export class TrackedPromise<Value = unknown> implements PromiseLike<Value> {
    constructor(existingPromise: Promise<Value>);
    constructor(callback: ConstructorParameters<typeof Promise<Value>>[0]);
    constructor(promiseOrCallback: /* ... */) { /* ... */ }

    // private, tracked, all state is derived from this,
    // since promises are not allowed to have more than one state. 
    #state: 
    | ['PENDING'] 
    | ['REJECTED', error: unknown] 
    | ['RESOLVED', value: Value];

    // private, the underlying promise that we're instrumenting
    #promise: Promise<Value>; 

    // upon success, Value, otherwise null.
    value: Value | null;

    // upon error, unknown, otherwise null.
    error: unknown;

    // state helpers
    isPending: boolean;
    isResolved: boolean;
    isRejected: boolean;

    // allows TrackedPromises to be awaited
    then(/* ... */): PromiseLike</* ... */>
}
```

Unlike [TrackedAsyncData][gh-tracked-async-data], all properties are accessible at throughout the the lifecycle of the promise. This is to reduce the pit-of-frustration and eliminate extraneous errors when working with promise-data. While ember-async-data can argue that it doesn't make sense to _even try_ accessing `value` until a promise is resolved, the constraint completely prevents UI patterns where you want value and loading state to be separate, and not part of a single whollistic if/else-if set of blocks. 

For simplicity, these new utilities will not be using `@dependentKeyCompat` to support the `@computed` era of reactivity.  pre-`@tracked` is before ember-source @ 3.13, which is from over 5 years ago, at the time of writing. For the broadest, most supporting libraries we have, 3.28+ is the supported range, and for speed of implementation, these tracked promise utilities can strive for the similar compatibility.

An extra feature that none of the previously mentioned implementations have is the ability to `await` directly. This is made easy by only implementing a `then` method -- and allows a good ergonomic bridge between reactive and non-reactive usages.


The implementation of `TrackedPromise` is intentionally limited, as we want to encourage reliance on [_The Platform_][mdn-Promise] whenever it makes sense, and is ergonomic to do so. For example, using `race` would still be done native, and can be wrapped for reactivity:
```js
/** @type {TrackedPromise} */
let trackedPromise 
    = trackPromise(Promise.race([promise1, promise2]));
```

[mdn-Promise]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise

### `@ember/reactive`

The process of making libraries support wide-ranges of `ember-source` is known. `ember-source` has recently been adapting its release process to use [release-plan][gh-release-plan], so that the [ember.js][gh-emberjs] repo can publish multiple packages seemslessly, rather than always bundle everything under one package.


With those new release capabilities within the [ember.js][gh-emberjs] repo, Instead of a polyfill for older versions of ember, `@ember/reactive`, the package (at the time of this RFC, does not exist, but would have the two exported utilities from it), would be pulished as its own package _and_ included with ember-source, as to not add more dependencies to the package.json going forward.

[gh-release-plan]: https://github.com/embroider-build/release-plan
[gh-emberjs]: https://github.com/emberjs/ember.js/


> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here. 

> Please keep in mind any implications within the Ember ecosystem, such as:
> - Lint rules (ember-template-lint, eslint-plugin-ember) that should be added, modified or removed
> - Features that are replaced or made obsolete by this feature and should eventually be deprecated
> - Ember Inspector and debuggability
> - Server-side Rendering
> - Ember Engines
> - The Addon Ecosystem
> - IDE Support
> - Blueprints that should be added or modified

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

> Keep in mind the variety of learning materials: API docs, guides, blog posts, tutorials, etc.

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

- reclaim the `ember` package and export undedr `ember/reactive`, add `ember` to the package.json.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
