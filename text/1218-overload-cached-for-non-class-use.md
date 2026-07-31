---
stage: accepted
start-date: 2026-07-30T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1218/
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

# Overload `cached` to work outside of classes 

## Summary

This RFC introduces an overload to the existing `cached` function, allowing it to be used outside of classes.

This is the caching companion to [RFC#1071](https://github.com/emberjs/rfcs/blob/master/text/1071-overload-tracked-for-non-class-use.md), which overloaded `tracked` for use outside of classes, and this RFC re-uses the interfaces defined there (adding `get` to `ReadOnlyReactive`, which was an oversight).

## Motivation

Our guides are gaining in-depth reactivity documentation ([ember-learn/guides-source#2219](https://github.com/ember-learn/guides-source/pull/2219)), and it's been useful to talk about reactive primitives as things _outside_ of classes, and compose/wrap them in to refactoring boundaries (classes, components, etc). 

[RFC#1071](https://github.com/emberjs/rfcs/blob/master/text/1071-overload-tracked-for-non-class-use.md) gave us `tracked()` for _root state_ outside of classes, but there is no ergonomic equivalent for caching a computation -- today, caching outside of a class requires either a class with a `@cached` getter, or dropping down to the caching primitives from [RFC#615](https://github.com/emberjs/rfcs/blob/master/text/0615-autotracking-memoization.md).

Enabling `cached` to be used outside of a class makes it a good tool for demos[^demos] for caching expensive computations in function-based APIs, such as _helpers_, _modifiers_, or _resources_ (or even in module space)[^apps]. They also provide a benefit in testing as well, since tests tend to want to assert that expensive computations do not re-run unnecessarily. 

`cached`-as-non-decorator was prototyped in [Starbeam](https://starbeamjs.com/guides/fundamentals/functions.html) (as `CachedFormula`). 

[^apps]: Apps typically should not have reactive state in module space, becaues it doesn't get automatically reset between tests, since we don't reload modules between each tests (partly for perf reasons). Cached state in module space is safer than root state (it recomputes when its inputs reset), but the same caution applies to what it reads.

[^demos]: demos _must_ over simplify to bring attention to a specific concept. Too much syntax getting in the way easily distracts from what is trying to be demoed. This has benefits for actual app development as well though, as we're, by focusing on concise demo-ability, gradually removing the amount of typing needed to create features. 

## Detailed design

Developers will continue to use: 

```ts
import { cached } from '@glimmer/tracking';
```

however, when called with a function (not used as a decorator), a different return value will be available:
- a `ReadOnlyReactive`

> [!IMPORTANT]
> This particular return value gives us the abilitiy in the future guides talking about reactivity a way to describe what the `@cached` decorator is doing (since decorators are not in every ecosystem), we can describe it as a syntactic sugar on top of a `ReadOnlyReactive`.

### Types 

This RFC re-uses the interfaces defined in [RFC#1071](https://github.com/emberjs/rfcs/blob/master/text/1071-overload-tracked-for-non-class-use.md):

~~~ts
interface Reactive<Value> {
   /**
    * The underlying value
    *
    * Allows easy usage of reactive values in templates.
    */
    value: Value;
}

// Useful internal concept for optimizations
interface ReadOnlyReactive<Value> extends Reactive<Value> {
    /**
    * The underlying value.
    * Cannot be set.
    */
    readonly value: Value;

    /**
    * Function short-hand of reading the value
    */
    get: () => Value;
}
~~~

`get` was an oversight in RFC#1071's `ReadOnlyReactive` -- this RFC adds it to the interface (rather than introducing a new interface). `TrackedValue` already has `get`, so it already conforms.

This RFC adds:

~~~ts
/**
* Utility to create a cached computation. 
*/
function cached<Value>(
    fn: () => Value,
    options?: { 
        description?: string 
    } = {}
): ReadOnlyReactive<Value>;
~~~

Unlike RFC#1071's `TrackedValue`, there is no `set`, `update`, or `freeze` -- the returned value has no storage of its own; its value comes entirely from the tracked state its function reads, so it is a `ReadOnlyReactive` from the start.

Behaviorally, `cached()` behaves almost the same as this function:
```js
import { createCache, getValue } from '@glimmer/tracking/primitives/cache';

function cached(fn, { description } = {}) {
  return new CachedValuePolyfill(fn, { description });
}

class CachedValuePolyfill {
    #cache;

    constructor(fn, options) {
        this.#cache = createCache(fn, options.description);
    }

    get value() {
        // reading entangles with the tracked state `fn` reads
        return getValue(this.#cache);
    }

    get() {
        return this.value;
    }
}
```

The function passed to `cached` is only re-invoked when tracked state it previously read has changed -- the same caching behavior as the `@cached` decorator from [RFC#566](https://github.com/emberjs/rfcs/blob/master/text/0566-memo-decorator.md).

### Usage

Caching a computation over module state.
This is already common in demos.

```gjs
import { tracked, cached } from '@glimmer/tracking';

const count = tracked(0);
const doubled = cached(() => count.value * 2);
const increment = () => count.value++;

<template>
    Count is: {{count.value}}, 
    doubled is: {{doubled.value}}

    <button {{on "click" increment}}>add one</button>
</template>
```

Using private mutable properties providing public, cached, read-only access:

```gjs
export class MyAPI {
    #state = tracked(0);

    #expensive = cached(() => veryExpensiveFunction(this.#state.value));

    get expensive() {
        return this.#expensive.value;
    }

    doTheThing() {
        this.#state.value = secretFunctionFromSomewhere(); 
    }
}
```

### Re-implementing `@cached` 

> [!NOTE]
> This is a conceptual exercise, and for performance reasons it won't be implemented this way

For most current ember projects, using the TC39 Stage 1 implementation of decorators:

```js
import { cached as glimmerCached } from '@glimmer/tracking';

function cached(target, key, { get }) {
  let caches = new WeakMap();

  function getCache(obj) {
    let cache = caches.get(obj);

    if (cache === undefined) {
      cache = glimmerCached(() => get.call(obj), { description: `cached:${key}` });
      caches.set(obj, cache);
    }

    return cache;
  };

  return {
    get() {
      return getCache(this).value;
    },
  };
}
```

<details><summary>Using spec / standards-decorators</summary>

```js
import { cached as glimmerCached } from '@glimmer/tracking';
    
export function cached(target, context) {
  let caches = new WeakMap();

  return function (this: object) {
    let cache = caches.get(this);

    if (cache === undefined) {
      cache = glimmerCached(() => target.call(this), { description: `cached:${String(context.name)}` });
      caches.set(this, cache);
    }

    return cache.value;
  };
}
```

</details>

## How we teach this

The `cached` function is a low-level tool, for folks that want specific behavior and for most real applications, folks should continue to use classes, with `@cached` getters, as the combination of classes with decorators provide unparalleled ergonomics in state management. 

However, developers may think of `@cached` (or decorators in general) as magic -- we can utilize `cached()` as a storytelling tool to demystify how `@cached` works -- since `cached()` will be public API, we can easily explain how `cached()` is used to _create the `@cached` decorator_ (without discussing the real private APIs that we _don't_ want folks using (such as those exported from `@glimmer/validator`).

We can even use the example over-simplified implementation of `@cached` from the _Detailed Design_ section above.

Together with `tracked()` from RFC#1071, this completes the story for teaching reactivity without classes. As with the `@cached` decorator, overuse is discouraged: most computations are cheap and should stay plain functions.

### When to use `value`

Allows for easy use in templates as well as in getters:

```gjs
import { tracked, cached } from '@glimmer/tracking';

const count = tracked(0);
const doubled = cached(() => count.value * 2);
const increment = () => count.value++;

<template>
    Count is: {{count.value}},
    doubled is: {{doubled.value}}

    <button {{on "click" increment}}>add one</button>
</template>
```

### When to use `get()`

Allows passing the read as a function, e.g. to utilities that accept a thunk:

```gjs
import { tracked, cached } from '@glimmer/tracking';

const count = tracked(0);
const doubled = cached(() => count.value * 2);

const logLater = (read) => setTimeout(() => console.log(read()));

<template>
    {{doubled.value}}

    <button {{on "click" (fn logLater doubled.get)}}>log it</button>
</template>
```

## Drawbacks

- same API does multiple things based on usage, but developers should be used to this somewhat as overloading is nothing new -- TS will also be agreeable with the overloads -- and RFC#1071 has already established this pattern for `tracked`
- potential confusion between `@cached` (decorates a getter) and `cached(fn)` (wraps a function) -- though the mental model is the same: "cache this computation against the tracked state it reads"

## Alternatives

- completetly new API, such as `memo` or `formula`
- continue pointing folks at `createCache` / `getValue` from `@glimmer/tracking/primitives/cache` (2 imports, and a `primitives` path that signals "not for app developers")

## Unresolved questions

- none yet

## Appendix

### Relationship to the primitives from RFC#615

The primitives from [RFC#615](https://github.com/emberjs/rfcs/blob/master/text/0615-autotracking-memoization.md) (`createCache` / `getValue`) provide the same caching capability, and the implementation of `cached()` sits on the same machinery. `cached()` packages that capability in the already-public `cached` import, without requiring 2 imports from a `primitives` path. The primitives remain as-is, and stay useful for library authors.

### Deferred: an `equals` option

An earlier draft proposed an `equals` option for retaining the previous value's identity when a re-computation produced an equivalent result. Since calling the function at all is the expensive part, re-running it only to discard the result has unclear benefit -- so it is not part of this RFC. It could be added later without changing the API proposed here.

### Naming: value

Consistent with RFC#1071's `TrackedValue`. Value is generic enough, and is a generally understood concept without nuance.

### Naming: `get` and `read` 

The same reasoning as RFC#1071 applies:

- `get` implies that you are always going to do something with what is given to you 
- `read` somewhat implies that you want to see the state of the cached value, but is ambigous about if you want to do anything with that information

`get`, in particular, (while ~ unfortunately ~, matches legacy naming in our history), matches existing JS concepts from Map, WeakMap, other other concepts.

### Why no `set`, `update`, or `freeze`

The value returned from `cached()` has no storage of its own -- its value is entirely a function of the tracked state its function reads. Writing to it is meaningless, and it is already permanently "frozen" from the consumer's point of view. This is also why `cached()` returns a `ReadOnlyReactive` rather than a `Reactive`.

### Extension

If folks wanted, they could make their own cached value with previous or historical values. This could be useful for extremely expensive operations that depend on previous computations. 
To do this, folks would need to implement their own class:
```js
class CachedValueWithHistory {
    #previous;
    #current;

    constructor(fn) {
        this.#current = cached(() => {
            this.#previous = untrack(() => this.#current?.value);
            return fn(this.#previous);
        });
    }

    get value() {
        return this.#current.value;
    }

    get previous() {
        return this.#previous;
    }

    // ...
}
```
