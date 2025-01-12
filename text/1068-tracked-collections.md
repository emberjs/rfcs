---
stage: accepted
start-date: 2025-01-12T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - data
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1068 
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

# Built in tracking utilities for common collections 

## Summary

This RFC proposes making the set of collections from `tracked-built-ins` built in to the framework, in a byte-wise opt-in in a brand new package (`@ember/reactive`).

Additionally, these APIs can unblock the implementation of [RFC#1000: Make array built-in in strict mode](https://github.com/emberjs/rfcs/pull/1000) and [RFC#999: make hash built in in strict mode](https://github.com/emberjs/rfcs/pull/999)

## Motivation

tl;dr:

- performance 
- discoverability
- aiming towards better cohesion 

Because `tracked-built-ins` is built on top of public APIs, in particular, `ember-tracked-storage-polyfill`, we can expect to gain performance benefits by implementing the tracked collections directly into the framework, as we can eliminate ~2 layers of abstraction/wrapping.


Additionally, `tracked-built-ins` not being built in to the framework, or properly documented in the ember guides has had some negative consequences on folks apps.

For example, this often-inefficient pattern of re-assigning the whole reference. 
```js
@tracked value = [];
addItem = (x) => {
    this.value = [...this.value, x];
}
```

For large sets of data, rendered in a list (often tables), this pattern causes unneeded work in the reactivity-system.

We now know that `tracked-built-ins`' `TrackedArray` would be a good way to only _append_ an item to the array, and thus append DOM to our UI, without the reactive system doing anything to the data that hasn't changed.

Some may argue that it's our renderer's responsibility to detect this situation, and optimize best it can, and while there are opportunities we can find to optimize rendering, we also can't make an assumption that _either_ re-assigning or tracked collection usage is going to be the most performant. Developers can measure in their own app.


This is outside the scope of this RFC, but for some underlying motivation,
another motivation is along the lines of [reigning in our imports](https://github.com/emberjs/rfcs/pull/1060#issuecomment-2557737145) over time, potentially by eventually reclaiming the `'ember'` package, so that there is a simple package.json that can be _the framework_, which aligns with real imports (or re-exports) so that we don't _require_ build-system gymnastics in order to build ember apps.

<details><summary>examples</summary>

Old:
```js
import Route from '@ember/routing/route';
import Service, { service } from '@ember/service';
import Component from '@glimmer/component';
import { tracked, cached } from '@glimmer/tracking';
```

New:
```js
import Route from 'ember/routing/route';
import Service, { service } from 'ember/service';
import Component from 'ember/glimmer';
import { tracked, cached } 'ember/reactive';
```

The details of this are absolutely up for debate -- this is just demonstrating the concept -- by re-exporting everything from a single package, it gives folks an opportunity to use ember without embroider -- and without any build at all. 

</details>

## Detailed design

while _most of this_ already implemented, here is the behavior we expect when using any tracked wrapper:

- all property accesses should "entangle" with that property 
- all property sets should "dirty" that property
- changes to the length, or overall collection, is represented by an invisible-to-users "collection" internal tracked property, so that iteration can be dirtied
- changes to a collection (add, insert, delete, etc) should cause iteration (each, each-in) to only render what changed 
- changes to a collection mutate the original passed in data, if any (which saves memory thrashing)
- deleting an entry in a collection should relieve memory pressure
- deleting an entry in a collection should dirty the "collection"
- prototype and `instanceof` checks should still work, e.g.: a `TrackedArray` should still return true from `Array.isArray`, and an instance of `TrackedSet` should be an `instanceof Set`.
- no `@dependentKeyCompat`, see: [`@ember-compat/tracked-built-ins`](https://www.npmjs.com/package/@ember-compat/tracked-built-ins)

How do we handle when the platform adds new APIs?

For example, Set has had new APIs added recentlny, and `tracked-built-ins` had to be updated to support those, so if possible, it would be ideal to rely on deferring to the underlying implementation as much as possible, rather than re-implementing a class-wrapper for all known methods -- proxies are particularly good at this -- and while folks have had complaints about proxies in the past, the user-facing API and underlying implementation of all these proxies would be the exact same, so the proxy isn't hiding anything.


### The import

object and array:
```js
import { 
    // Our existing utilities
    TrackedObject, TrackedArray,
    // able to be used in templates, no 'new'
    trackedObject, trackedArray, 
} from '@ember/reactive';
```

maps and sets:
```js
import { 
    // Our existing utilities
    TrackedMap, TrackedWeakMap,
    TrackedSet, TrackedWeakSet
    // able to be used in templates, no 'new'
    trackedMap, trackedWeakMap,
    trackedSet, trackedWeakSet
} from '@ember/reactive/collections';
```


### `trackedObject`, `trackedArray`, `trackedMap`, etc

These utilities wrap the call to their respective constructors. For example, for `trackedObject`, the implementation and type declaration may look like this:

```ts
export function trackedObject<Value>(data?: Value): NonNullable<Value> {
    return new TrackedObject(data);
}
```

Some examples assuming implementation of [RFC#998: Make fn built-in in strict-mode](https://github.com/emberjs/rfcs/pull/998) as well as [RFC#997: Make on built-in in strict-mode](https://github.com/emberjs/rfcs/pull/997):

#### Example `trackedArray`


```gjs
import { trackedArray } from '@ember/reactive';

const nonTrackedArray = [1, 2, 3];
const addTo = (arr) => arr.push(Math.random());

<template>
    {{#let (trackedArray nonTrackedArray) as |arr|}}
        {{#each arr as |datum|}}
            {{datum}}
        {{/each}}

        <button {{on 'click' (fn addTo arr)}}>Add Item</button> 
    {{/let}}
</template>
```

> [!NOTE]  
> Since [RFC#1000: Make Array built-in in strict mode](https://github.com/emberjs/rfcs/pull/1000) is stalled due to the original implementation of `(array)` being underspecified, the new implementation of the built in `(array)` could use this `trackdArray` implementation instead of re-defining the specification of how `(array)` works -- and this new implementation would probably more align with how folks expect `(array)` to work.

With RFC#1000, the above example would be behaviorally equivalent to:
```gjs
const nonTrackedArray = [1, 2, 3];
const addTo = (arr) => arr.push(Math.random());

<template>
    {{#let (array nonTrackedArray) as |arr|}}
        {{#each arr as |datum|}}
            {{datum}}
        {{/each}}

        <button {{on 'click' (fn addTo arr)}}>Add Item</button> 
    {{/let}}
</template>
```

#### Example `trackedObject`

```gjs
import { trackedObject } from '@ember/reactive';

const nonTrackedObject = { a: 1 };
const addTo = (obj) => obj[Math.random()] = Math.random();

<template>
    {{#let (trackedObject nonTrackedObject) as |obj|}}
        {{#each-in obj as |key value|}}
            <pre>{{globalThis.JSON.stringify obj null 3}}</pre>
        {{/each-in}}

        <button {{on 'click' (fn addTo obj)}}>Add Pair</button>
    {{/let}}
</template>
```

> [!NOTE]  
> Since [RFC#999: Make hash built-in in strict mode](https://github.com/emberjs/rfcs/pull/999) is stalled due to the original implementation of `(hash)` being underspecified, the new implementation of the built in `(hash)` could use this `trackedObject` implementation instead of re-defining the specification of how `(hash)` works -- and this new implementation would probably more align with how folks expect `(hash)` to work.


With RFC#999, the above example would be behaviorally equivalent to:

```gjs
const nonTrackedObject = { a: 1 };
const addTo = (obj) => obj[Math.random()] = Math.random();

<template>
    {{#let (hash nonTrackedObject) as |obj|}}
        {{#each-in obj as |key value|}}
            <pre>{{globalThis.JSON.stringify obj null 3}}</pre>
        {{/each-in}}

        <button {{on 'click' (fn addTo obj)}}>Add Pair</button>
    {{/let}}
</template>
```

### `@ember/reactive`

The process of making libraries support wide-ranges of `ember-source` is known. `ember-source` has recently been adapting its release process to use [release-plan][gh-release-plan], so that the [ember.js][gh-emberjs] repo can publish multiple packages seemslessly, rather than always bundle everything under one package.

With those new release capabilities within the [ember.js][gh-emberjs] repo, Instead of a polyfill for older versions of ember, `@ember/reactive`, the package (at the time of this RFC, does not exist, but would have the two exported utilities from it), would be published as its own `type=module` package _and_ included with ember-source, as to not add more dependencies to the package.json going forward.

[gh-release-plan]: https://github.com/embroider-build/release-plan
[gh-emberjs]: https://github.com/emberjs/ember.js/

Why `type=module`?

This is a requirement for some optimization features of packages (webpack / vite), such as _proper_ treeshaking -- without `type=module`, the best optimization we can get is "pay for only what you import". For large projects this isn't so much of a problem, but for small projects (or highly optimized projects), the impact to network transfer/parse/eval is measurable. This RFC is also proposing that `@ember/reactive` be _the_ place for all our ecosystem's reactivity utilities will end up once they've been proven out, tested, and desire for standardization is seen.

For example, other future exports from `@ember/reactive` (in future RFCs), may include:
- Resource
- AsyncResource
- TrackedPromise
- localCopy
- certain [window properties](https://svelte.dev/docs/svelte/svelte-reactivity-window)
- ...and more

without the static analysis guarantees of `type=module`, every consumer of `@ember/reactive` would always have all of these exports in their build.
For some utilities, we can place them under sub-path-exports, such as `@ember/reactive/window`, for window-specific reactive properties, but the exact specifics of each of these can be hashed out in their individual RFCs.


### Consumption

When a project wants to use `@ember/reactive`, they would then only need to install the package separately / add it to their `package.json`.

The proposed list of compatibility here is only meant as an example -- if implementation proves that more can be supported easier, with less work, that should be pursued, and this part is kind of implementation detail.

But for demonstration:
- apps pre [version available], would add `@ember/reactive` to their `devDependencies` or `dependencies`
  - importing `@ember/reactive` would be handled by ember-auto-import/embroider (as is the case with all v2 addons)
- v1 addons would not be supported
- v2 addons, for maximum compatibility, would need to add `@ember/reactive` to their `dependencies`
  - in consuming apps post [version available], this would be optimized away if the version declared in dependencies satisfies the range provided by the consuming app (an optimization that packagers already do, and nothing we need to worry about)
- apps post [version available], would not need to add `@ember/reactive` to their `devDependencies` or `dependencies`, as we can rely on the `ember-addon#renamed-modules` config in ember-source's `package.json`.

## How we teach this

### API Docs

Most of the API docs are already written in `tracked-built-ins`, so we can re-use those. 

We do have new template-oriented helpers tho (not requiring `new`), and it is worth showing how to use those.

#### `trackedArray`

```gjs
import { trackedArray } from '@ember/reactive';
import { on } from '@ember/modifier';
import { fn } from '@ember/helper';

const nonTrackedArray = [1, 2, 3];
const addTo = (arr) => arr.push(Math.random());

<template>
    {{#let (trackedArray nonTrackedArray) as |arr|}}
        {{#each arr as |datum|}}
            {{datum}}
        {{/each}}

        <button {{on 'click' (fn addTo arr)}}>Add Item</button> 
    {{/let}}
</template>
```

#### `trackedObject`

```gjs
import { trackedObject } from '@ember/reactive';
import { on } from '@ember/modifier';
import { fn } from '@ember/helper';

const nonTrackedObject = { a: 1 };
const addTo = (obj) => obj[Math.random()] = Math.random();

<template>
    {{#let (trackedObject nonTrackedObject) as |obj|}}
        {{#each-in obj as |key value|}}
            <pre>{{globalThis.JSON.stringify obj null 3}}</pre>
        {{/each-in}}

        <button {{on 'click' (fn addTo obj)}}>Add Pair</button>
    {{/let}}
</template>
```

### Guides

Existing places that import from `tracked-built-ins` would update to the new imports -- no other changes would be needed.

- [This page](https://guides.emberjs.com/release/configuring-ember/disabling-prototype-extensions/#toc_tracking-of-changes-in-arrays) needs to be updated as `@glimmer/tracking` doesn't have `TrackedArray` today.

Something that could be used today, and definitely should be added is a page on how to handle referential integrity. Most of the "tracked" guides only touch on tracking _references_ (via `@tracked`). For example, each of `TrackedArray`, `TrackedMap`, `TrackedSet`, etc can be used in these ways:

#### Static reference

```js
class Demo {
    collection = new TrackedMap();
}
```

Changes to `this.collection` can only happen via `Map` methods.

#### Double static reference

```js
class Demo {
    @tracked collection = new TrackedMap();
}
```
Changes to `this.collection` can happen via `Map` methods, as well as replacing the entirely collection can occur via re-assigning `this.collection` to a brand new `TrackedMap`. This also has a potential performance hazard, of re-assigning `this.collection` to a clone of the `TrackedMap`.

#### Based on Args

```js
class Demo extends Component {
    @cached
    get collection() {
        return new TrackedMap(this.args.otherData);
    }
}
```
Changes to the collection can happen via `Map` methods, as well as changes to `@otherData` will cause the entirety of `this.collection` to be re-created, with the previous instance being garbage collected. Usage of `@cached` is important here, because repeat-accesses to `this.collection` would otherwise create completely unrelated `TrackedMap`s -- i.e.: Updating a `TrackedMap` would have no effect on a `TrackedMap` read elsewhere as they are different instances.  


## Drawbacks

- A migration
  - however, the migration is completely optional as `tracked-built-ins` would still exist. The benefit to this RFC is for new projects, and apps that care more about performance.

## Alternatives

- reclaim the `ember` package and export under `ember/reactive`, add `ember` to the package.json.
  - doing this _would_ require a polyfill, as `ember` is already available in all versions of projects, but it does not have sub-path-exports that folks use.
- use `/reactivity` instead of `/reactive`
- re-use `@glimmer/tracking`
  - would require that `@glimmer/tracking` move in to the `ember-source` repo
  - would also require a polyfill, as prior versions of `@glimmer/tracking` would not have the new behaviors
  - there is an existing typo in the guides that hints at using this already for `TrackedArray`

## Unresolved questions

none (yet)