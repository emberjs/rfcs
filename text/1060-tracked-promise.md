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
  accepted: https://github.com/emberjs/rfcs/pull/1060 
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

For loading UIs, there are concerns that this proposed utility will not be concerned with directly, but also the utility still needs to enable usage of those UIs, so in _How we teach this_, there will be examples of various loading patterns and re-implementations of existing utilities (such as those mentioned in the _Motivation_ section above) using the new tracking utility for promises.


The import:
```js
import { TrackedPromise, trackPromise } from '@ember/reactive';
```

> [!NOTE]
> Key behaviors:
> - if the passed promise is resolved, or a non-promise, we do not await, this allows 
>   values to not block render (or cause a re-render if they don't need to)
> - no `@dependentKeyCompat`
> - promise states are all mutually exclusive

### `trackPromise`

This utility wraps and instruments any promise with reactive state, `TrackedPromise`.  


Sample type declaration
```ts
export function trackPromise<Value>(
    existingPromise: Promise<Value> | Value
): TrackedPromise<Value> {
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


#### Subtle Notes

If a promise is passed to `trackPromise` or `TrackedPromise` multiple times, we don't want to _re-do_ any computations.

Examples:

```gjs
let a = Promise.resolve(2); // <state> "fulfilled"

<template>
  {{#let (trackPromise a) as |state|}}
    {{state.value}}
  {{/let}}

  {{#let (trackPromise a) as |state|}}
    {{state.value}}
  {{/let}}
</template>
```

This component renders only once, and _both_ occurances of of `trackPromise` immediately resolve and _never_ enter the pending states.



```gjs
let a = Promise.resolve(2); // <state> "fulfilled"
let b = Promise.resolve(2); // <state> "fulfilled"

<template>
  {{#let (trackPromise a) as |state|}}
    {{state.value}}
  {{/let}}

  {{#let (trackPromise b) as |state|}}
    {{state.value}}
  {{/let}}
</template>
```

In this component, it _also_ only renders once as both promises are resolved, and we can adapt the initial state returned by `trackPromise` to reflect that.



### `@ember/reactive`

The process of making libraries support wide-ranges of `ember-source` is known. `ember-source` has recently been adapting its release process to use [release-plan][gh-release-plan], so that the [ember.js][gh-emberjs] repo can publish multiple packages seemslessly, rather than always bundle everything under one package.

With those new release capabilities within the [ember.js][gh-emberjs] repo, Instead of a polyfill for older versions of ember, `@ember/reactive`, the package (at the time of this RFC, does not exist, but would have the two exported utilities from it), would be pulished as its own `type=module` package _and_ included with ember-source, as to not add more dependencies to the package.json going forward.

[gh-release-plan]: https://github.com/embroider-build/release-plan
[gh-emberjs]: https://github.com/emberjs/ember.js/

Why `type=module`?

This is a requirement for some optimization features of packages (webpack / vite), such as _proper_ treeshaking -- without `type=module`, the best optimization we can get is "pay for only what you import". For large projects this isn't so much of a problem, but for small projects (or highly optimized projects), the impact to network transfer/parse/eval is measurable. This RFC is also proposing that `@ember/reactive` be _the_ place for all our ecosystem's reactivity utilities will end up once they've been proven out, tested, and desire for standardation is seen.

For example, other future exports from `@ember/reactive` (in future RFCs), may include:
- TrackedObject
- TrackedArray
- TrackedMap
- TrackedSet
- TrackedWeakSet
- TrackedWeakMap
- localCopy
- certain [window properties](https://svelte.dev/docs/svelte/svelte-reactivity-window)
- ...and more

without the static analysis guarantees of `type=module`, every consumer of `@ember/reactive` would always have all of these exports in their build.
For some utilities, we can place them under sub-path-exports, such as `@ember/reactive/window`, for window-specific reactive properties, but the exact specifics of each of these can be hashed out in their individual RFCs.


### Consumption

When a project wants to use `@ember/reactive`, they would then only need to install the package separately / add it to their `package.json`.

The proposed list of compatibilyt here is only meant as an example -- if implementation proves that more can be supported easier, with less work, that should be pursued, and this part is kind of implementation detail.

But for demonstration:
- apps pre [version available], would add `@ember/reactive` to their `devDependencies` or `dependencies`
  - importing `@ember/reactive` would be handled by ember-auto-import/embroider (as is the case with all v2 addons)
- v1 addons would not be supported
- v2 addons, for maximum compatibility, would need to add `@ember/reactive` to their `dependencies`
  - in consuming apps post [version available], this would be optimized away if the version declared in dependencies satisfies the range provided by the consuming app (an optimization that packagers already do, and nothing we need to worry about)
- apps post [version available], would not need to add `@ember/reactive` to their `devDependencies` or `dependencies`, as we can rely on the `ember-addon#renamed-modules` config in ember-source's `package.json`.

## How we teach this

### API Docs

#### `trackPromise`

```js
import { trackPromise } from '@ember/reactive';
```

The returned value is an instance of `TrackedPromise`, and is for instrumenting promise state with reactive properties, so that UI can update as the state of a promise changes over time.

When a non-promise is passed, as one may do for a default value, it'll behave as if it were a resolved promise, i.e.: `Promise.resolve(passedValue)`.

This is a shorthand utility for passing an existing promise to `TrackedPromise`.


Example in a template-only component
```gjs
import { trackPromise } from '@ember/reactive';

function wait(ms) {
  return new Promise((resolve) => setTimeout(resolve, ms));
}

<template>
  {{#let (trackPromise (wait 500)) as |state|}}
    isPending:  {{state.isPending}}<br>
    isResolved: {{state.isResolved}}<br>
    isRejected: {{state.isRejected}}<br>
    value:      {{state.value}}<br>
    error:      {{state.error}}<br>
  {{/let}}
</template>
```

Example in a class component:
```gjs
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { trackPromise } from '@ember/reactive';

export default class Demo extends Component {
  @cached
  get state() {
    // promise resolves after 400ms
    let promise = new Promise((resolve) => {
      setTimeout(resolve, 400);
    });

    return trackPromise(promise);
  }

  <template>
    isPending:  {{this.state.isPending}}<br>
    isResolved: {{this.state.isResolved}}<br>
    isRejected: {{this.state.isRejected}}<br>
    value:      {{this.state.value}}<br>
    error:      {{this.state.error}}<br>
  </template>
}
```


#### `TrackedPromise`

```js 
import { TrackedPromise } from '@ember/reactive';
```

Creates a tracked `Promise`, with `tracked` properties for implementing UI that updates based on the state of a promise.

When a non-promise is passed, as one may do for a default value, it'll behave as if it were a resolved promise, i.e.: `Promise.resolve(passedValue)`.

Creating a tracked promise from a non-async API:
```gjs
import { TrackedPromise } from '@ember/reactive';

function wait(ms) {
  return new TrackedPromise((resolve) => setTimeout(resolve, ms));
}

<template>
  {{#let (wait 500) as |state|}}
    isPending:  {{state.isPending}}<br>
    isResolved: {{state.isResolved}}<br>
    isRejected: {{state.isRejected}}<br>
    value:      {{state.value}}<br>
    error:      {{state.error}}<br>
  {{/let}}
</template>
```

Creating a tracked promise from an existing promise:

```gjs
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { TrackedPromise } from '@ember/reactive';

export default class Demo extends Component {
  @cached
  get state() {
    let id = this.args.personId;
    let fetchPromise = 
      fetch(`https://swapi.tech/api/people/${id}`)
        .then(response => response.json());

    return new TrackedPromise(fetchPromise);
  }

  <template>
    ... similar usage as above 
    {{this.state.isPending}}, etc
  </template>
}
```

### Guides

#### use with `fetch`

With `@cached`, we can make any getter have stable state and referential integrity, which is essential for having multiple accesses to the getter return the same object -- in this case, the return value from `trackPromise`:

```gjs
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { trackPromise } from '@ember/reactive';

export default class Demo extends Component {
  @cached
  get requestState() {
    let id = this.args.personId;
    let fetchPromise = 
      fetch(`https://swapi.tech/api/people/${id}`)
        .then(response => response.json());

    return trackPromise(fetchPromise);
  }

  // Properties can be aliased like any other tracked data
  get isLoading() {
    return this.requestState.isPending;
  }

  <template>
    {{#if this.isLoading}}
       ... loading ...
    {{else if this.requestState.value}}
       <pre>{{globalThis.JSON.stringify this.requsetState.value null 2}}</pre>
    {{else if this.requestState.error}}
       oh no!
       <br>
       {{this.requestState.error}}
    {{/if}}
  </template>
}
```
In this example, we only ever show one of the states at a time, loading _or_ the value _or_ the error.
We can separate each of these, if desired, like this:
```gjs
<template>
  {{#if this.isLoading}}
      ... loading ...
  {{/if}}
  
  {{#if this.requestState.value}}
      <pre>{{globalThis.JSON.stringify this.requsetState.value null 2}}</pre>
  {{/if}}

  {{#if this.requestState.error}}
      oh no!
      <br>
      {{this.requestState.error}}
  {{/if}}
</template>
```
Doing so would allow more separation of loading / error UI, such as portaling loading / error notifications to somewhere central in your applications.

NOTE: using `@cached` with promises does not enable cancellation, as there is no _lifetime_ to attach to at the getter/property level of granularity.[^resources] 

[^resources]: This is where _resources_ can help (topic for later) -- otherwise you need to use a full component just for destruction (or `invokeHelper` on a class-based helper, which is very un-ergonomic).

#### creating reactive promises

We can use `TrackedPromise` to turn not-async APIs into reactive + async behaviors -- for example, if we want to make a promise out of `setTimeout`, and cause an artificial delay / timer behavior:

```gjs
import Component from '@glimmer/component';
import { cached } from '@glimmer/tracking';
import { TrackedPromise } from '@ember/reactive';

export default class Demo extends Component {
  @cached
  get () {
    return new TrackedPromise((resolve => {
       setTimeout(() => {
        resolve();
       }, 5_000 /* 5 seconds */);
    }));
  }

  get showSubscribeModal() {
    return this.requestState.isResolved;
  }

  <template>
     {{#if this.showSubscribeModal}}
        <dialog open>
          Subscribe now!

          ...
        </dialog>
     {{/if}}
  </template>
}
```

#### keeping the latest value while new content loads

This pattern is useful for dropdown searches, tables (filtering), and other UIs which could otherwise cause large layout shifts / excessive repaints.

We still want a loading UI on _initial data load_, but want to have a more subtle subsequent loading indicator (yet still prominently visible)

```gjs
import Component from '@glimmer/component';
import { tracked, cached } from '@glimmer/tracking';
import { trackPromise } from '@ember/reactive';
import { isEmpty } from '@ember/utils';

export default class Demo extends Component {
  @tracked id = 51;
  updateId = (event) => this.id = event.target.value;

  @cached
  get request() {
    let promise = fetch(`https://swapi.tech/api/peopile/${this.id}`)
      .then(response => respones.json());

    return trackPromise(promise);
  }

  #previous;
  #initial = true;

  @cached
  get latest() {
    let { value, isPending } = this.request;

    if (isPending) {
      if (this.#previous === undefined && this.#initial) {
        this.#initial = false;

        return value;
      }

      return (this.#previous = isEmpty(value) ? this.#previous : value);
    }

    return this.#previous = value;
  }

  <template>
    <label>
      Person ID
      <input type='number' value={{this.id}} {{on 'input' this.updateId}} >
    </label>

    {{! initially, there is no latest value as the first request is still loading}}
    {{#if this.latest}}

      <div class="async-state">
        {{! Async state for subsequent requests, only}}
        {{#if this.request.isPending}}
            ... loading ...
        {{else if this.request.isRejected}}
            error!
        {{/if}}
      </div>

      {{! pretty print the response as JSON }}
      <pre>{{globalThis.JSON.stringify this.latest null 2}}</pre>

    {{else}}
      {{! This block only matters during the initial request }}

      {{#if this.request.isRejected}}
        error loading initial data!
      {{else}}
        <pre> ... loading ... </pre>
      {{/if}}

    {{/if}}
  </template>
}
```

## Drawbacks

I think not doing this has more drawbacks than doing it. A common problem we have is that we have too many packages and too many ways to do things. Our users long for "the ember way" to do things, and a comprehensive reactive library full of vibrant, shared utilities is one such way to bring back some of what folks are longing for.

## Alternatives

- reclaim the `ember` package and export under `ember/reactive`, add `ember` to the package.json.
  - doing this _would_ require a polyfill, as `ember` is already available in all versions of projects, but it does not have sub-path-exports that folks use.
- use `/reactivity` instead of `/reactive`
- re-use `@glimmer/tracking`
  - would require that `@glimmer/tracking` move in to the `ember-source` repo
  - would also require a polyfill, as prior versions of `@glimmer/tracking` would not have the new behaviors
- whole-sale pull in parts of `@warp-drive/ember` (though, much of this RFC is already heavily influenced by their `getPromiseState`, which is backed by a promise cache -- all of which is sort of an implementation detail as far this RFC is concerned)

## Unresolved questions

none (yet)
