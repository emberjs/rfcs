---
stage: ready-for-release
start-date: 2025-01-19T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1071'
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

# Overload `tracked` to work outside of classes 

## Summary

This RFC introduces two overloads to the existing `tracked` function, allowing it to be used outside of classes.

## Motivation

Our documentation / guides currently don't have much of anything on reactivity, and over the years, it's been useful to talk about reactive primitives as things _outside_ of classes, and compose/wrap them in to refactoring boundaries (classes, components, etc). 

Additionally, other new reactive utilities have an options object that allow power-users to configure equality and dirtying behavior (see: `@ember/reactive/collections` for those), and we want that same configurability for `tracked`.

Enabling `tracked` to be used outside of a class makes it a good tool for demos[^demos] for creating reactive values in function-based APIs, such as _helpers_, _modifiers_, or _resources_ (or even in module space)[^apps]. They also provide a benefit in testing as well, since tests tend to want to work with some state. 

This is not too dissimilar to the [Tracked Storage Primitive in RFC#669](https://github.com/emberjs/rfcs/blob/master/text/0669-tracked-storage-primitive.md)[^tracked-storage-future]. Making `tracked` work outside of classes provides the same benefit without requiring 3 imports to use. This RFC intends to provide a tool enabling us to sunset/abandon the unimplemented storage-primitives. 

`tracked`-as-non-decorator was prototyped in [Starbeam](https://starbeamjs.com/guides/fundamentals/cells.html) and has been available for folks to try out in ember via [ember-resources](https://github.com/NullVoxPopuli/ember-resources/tree/main/docs/docs) (though, under a different name: `cell`). 

[^apps]: Apps typically should not have reactive state in module space, becaues it doesn't get automatically reset between tests, since we don't reload modules between each tests (partly for perf reasons).

[^demos]: demos _must_ over simplify to bring attention to a specific concept. Too much syntax getting in the way easily distracts from what is trying to be demoed. This has benefits for actual app development as well though, as we're, by focusing on concise demo-ability, gradually removing the amount of typing needed to create features. 

[^tracked-storage-future]: This may likely be deprecated at some point -- especially since it was never implemented, and `cell`/`tracked` is a better successor to it (in general). 

## Detailed design

Developers will continue to use: 

```ts
import { tracked } from '@glimmer/tracking';
```

however, when used as a function (not a decorator), a different return value will be available:
- a `Reactive`

> [!IMPORTANT]
> This particular return value gives us the abilitiy in the future guides talking about reactivity a way to describe what the `@tracked` decorator is doing (since decorators are not in every ecosystem), we can describe them as a syntactic sugar on top of a `Reactive`.

### Types 

Some interfaces to share with future low-level reactive primitives:

~~~ts
interface Reactive<Value> {
   /**
    * The underlying value
    *
    * Allows easy usage of reactive values in templates.
    *
    * @example
    *
    * ```gjs
    * const myValue = tracked(0);
    * <template>
    *   {{myCell.value}}
    * </template>
    * ```
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
}

/**
* Utility to create a tracked value. 
*/
function tracked<Value>(
    initialValue: Value,
    options?: { 
        equals: (a: Value, b: Value) => boolean, 
        description?: string 
    } = {}
) {
  return new TrackedValue(
    initialValue,
    {
      equals: options?.equals ?? Object.is,
      description: options?.description
    }
  );
}

interface TrackedValue<Value> extends Reactive<Value> {
    /**
    * Function short-hand of updating the value
    * of the TrackedValue
    *
    * This is a convience method for different usage-styles, and is functionally the same as
    * assigning the `.value` value.
    */
    set: (value: Value) => boolean;

    /**
    * Function short-hand of reading the value
    * of the TrackedValue
    */
    get: () => Value;

    /**
    * Function short-hand for using the value to 
    * update the state of the TrackedValue
    */
    update: (fn: (value: Value) => Value) => void;

    /**
     * Prevents further updates, making the TrackedValue
     * behave as a ReadOnlyReactive
     *
     * This is an optimization that avoids update-checking later.
     */
    freeze: () => void;
}
~~~

Behaviorally, `tracked()` behaves almost the same as this function:
```js
function tracked(initial, { equals, description ) = {}) {
  return new TrackedValuePolyfill(initial, { equals, description });
}

class TrackedValuePolyfill {
    #isFrozen = false;
    #value;

    constructor(initialValue, options) {
        this.#value = initialValue;
        this.#equals = options.equals;
        this.#description = options.description;
        // ...
    }

    get value() {
        // + consume
        return this.#value;
    }
    set value(value) {
        assert(`Cannot set a frozen Cell`, !this.#isFrozen);
        // + dirty
        this.#value = value;
    } 

    get() {
        return this.value;
    }

    set(value) {
        this.value = value;
    }

    update(updater) {
        // #value is not tracked, 
        // so update reads without consuming
        this.value = updater(this.#value);
    }

    freeze() {
        this.#isFrozen = true;
    }
}
```

The key difference is that with a `TrackedValue`, we've exposed a new way for developers to decide when their value becomes dirty.
The above example, and the default value, would use the "always dirty" behavior of `() => false`.

This default value allows the `TrackedValue` to _conceptually_ (but not reality, becuse less abstraction layers are speedier) be the backing implementation of `@tracked`, as `@tracked` values do not have equalty checking to decide when to become dirty.

For example, with this `TrackedValue` and equality function:

```gjs
const value = tracked(0, { equals: (a, b) => a === b });

const selfAssign = () => value.value = value.value;

<template>
    <output>{{value}}</output>

    <button {{on 'click' selfAssign}}>Click me</button> 
</template>
```

The contents of the `output` element would never re-render due to the value never changing. 

This differs from `@tracked` (no args), as the contents of `output` would always re-render.

But, this RFC is always proposing we overload the decorator so that it can be configured to have customizable equality checking (see Usage below)


### Usage

Incrementing a count with local state.

```gjs
import { tracked } from '@glimmer/tracking';

const increment = (c) => c.value++;

<template>
    {{#let (tracked @initialCount) as |count|}}
        Count is: {{count.value}} 

        <button {{on "click" (fn increment count)}}>add one</button>
    {{/let}}
</template>
```

Incrementing a count with module state.
This is already common in demos.

```gjs
import { tracked } from '@glimmer/tracking';

const count = tracked(0);
const increment = () => count.value++;

<template>
    Count is: {{count.value}} 

    <button {{on "click" increment}}>add one</button>
</template>
```

Using private mutable properties providing public read-only access:

```gjs
export class MyAPI {
    #state = tracked(0);

    get myValue() {
        return this.#state.value;
    }

    doTheThing() {
        this.#state.value = secretFunctionFromSomewhere(); 
    }
}
```

Customizable equality in a class:

```gjs
import { tracked } from '@glimmer/tracking';

export default class Demo extends Component {
    @tracked({ equals: (a, b) => a === b}) value;

    // Does not "dirty" the value
    doNothing = () => this.value = this.value;

    // ....
}
```

when not called with options, the behavior remains the same as it does today:

```gjs
import { tracked } from '@glimmer/tracking';

export default class Demo extends Component {
    @tracked value;

    // The value *is* dirtied 
    doNothing = () => this.value = this.value;

    // ....
}
```


### Re-implementing `@tracked` 

> [!NOTE]
> This is a conceptual exercise, and for performance reasons it won't be implemented this way

For most current ember projects, using the TC39 Stage 1 implementation of decorators:

```js
import { tracked as glimmerTracked } from '@glimmer/tracking';

function tracked(target, key, { initializer }) {
  let caches = new WeakMap();

  function getValue(obj) {
    let myValue = caches.get(obj);

    if (myValue === undefined) {
      myValue = glimmerTracked(initializer.call(this), { equals: () => false, description: `tracked:${key}` });
      caches.set(this, myValue);
    }

    return myValue;
  };

  return {
    get() {
      return getValue(this).value;
    },

    set(value) {
      getValue(this).set(value);
    },
  };
}
```

<details><summary>Using spec / standards-decorators</summary>

```js
import { tracked as glimmerTracked } from '@glimmer/tracking';
    
export function tracked(target, context) {
  const { get } = target;

  return {
    get() {
      return get.call(this).value;
    },

    set(value) {
      get.call(this).set(value);
    },

    init(value) {
      return glimmerTracked(value, { equals: () => false, description: `tracked:${key}` });
    },
  };
}
```

</details>

### Usage as "side-signals"

Side-signals are the reactive approach used in tracked-bulit-ins, and other reactive-collections, where the Signal/tracked value isn't relevant / not used, and the reactive collection only uses the reactive structure for telling the renderer when to update/render/etc.

This can often be done with proxies:
```ts
import { tracked } from '@glimmer/tracking';

const nonReactiveObject = {};

let cache = new WeakMap<object, Map<key, TrackedValue>>();

function getReactivity(obj) {
    if (cache.has(obj)) {
        return reactiveCache.get(obj);
    }

    let cacheForObj = new Map();
    reactiveCache.set(obj, cacheForObj);
    return cacheForObj;
}

function consume(cacheForObj, key) {
    let cache = keyCache(cacheForObj, key);

    cache.get();
}

function dirty(cacheForObj, key) {
    let cache = keyCache(cacheForObj, key);

    cache.set(null);
}

function keyCache(obj, key) {
    let existing = cacheForObj.get(key);

    if (existing) {
        return existing;
    }

    existing = tracked(null, { equals: () => false });
    cacheForObj.set(existing, key);
    return existing;
}

const myReactiveObject = new Proxy(nonReactiveObject, {
    get(target, key) {
        let reactiveValues = getReactivity(target);

        consume(reactiveValues, key);

        return Refluct.get(...arguments);
    },
    set(target, key, value) {
        let reactiveValues = getReactivity(target);

        dirty(reactiveValues, key);

        return Refluct.set(...arguments);
    }
});

// now using `myReactiveObject` is reactive
myReactiveObject.foo = 2; 
```


## How we teach this

The `tracked` function is a low-level tool, for folks that want specific behavior and for most real applications, folks should continue to use classes, with `@tracked`, as the combination of classes with decorators provide unparalleled ergonomics in state management. 

However, developers may think of `@tracked` (or decorators in general) as magic -- we can utilize `tracked()` as a storytelling tool to demystify how `@tracked` works -- since `tracked()` will be public API, we can easily explain how `tracked()` is used to _create the `@tracked` decorator_ (without discussing the real private APIs that we _don't_ want folks using (such as those exported from `@glimmer/validator`).

We can even use the example over-simplified implementation of `@tracked` from the _Detailed Design_ section above.

### When to use `value`

Allows for easy use in templates as well as assignment

```gjs
import { tracked } from '@glimmer/tracking';

const increment = (c) => c.value++;

<template>
    {{#let (tracked @initialCount) as |count|}}
       Count is: {{count.value}} 

        <button {{on "click" (fn increment count)}}>add one</button>
    {{/let}}
</template>
```

### When to use `set()`

Allows partial application, e.g.:

```gjs
import { tracked } from '@glimmer/tracking';

<template>
    {{#let (tracked @initialCount) as |count|}}
       Count is: {{count.value}} 

        <button {{on "click" (fn count.set 1)}}>set one</button>
        <button {{on "click" (fn count.set 2)}}>set two</button>
    {{/let}}
</template>
```

### When to use `update()`

Allows updating a value without consuming/entangling with the current value.
Can be useful for changing the initial value once.


```gjs
import { tracked } from '@glimmer/tracking';

let num = tracked(0);

export class Demo extends Component {
    constructor() {
        super(...arguments);

        num.update((previous) => previous + 1);
    }

    <template>
        {{num.value}} => renders 1
    </template>
}

```


### When to use `freeze()`

Prevents further updates to the `ReactiveValue`. 

## Drawbacks

- same API does multiple things based on usage, but developers should be used to this somewhat as overloading is nothing new -- TS will also be agreeable with the overloads 
- potential confusion around why default `@tracked` has dirty-all-the-time reactivity (`equals: () => false`) -- this is for historical reasons, and we probably can't change this default behavior right now -- but might be able to in the future via different re-export from a different location (such as `@ember/reactive` -- but this has its own drawbacks (teaching, cognitive overload, etc))

## Alternatives

- completetly new API, such as `cell`

## Unresolved questions

- none yet

## Appendix

### Naming: value

Other proposed options from comments were: `current`.

Value is generic enough, and is a generally understood concept without nuance.


### Naming: `get` and `read` 

> [!NOTE]
> Developers should only access values they want to use, and not with the intent of causinge side-effects -- however, certain patterns (such as "side-signals")

- `get` implies that you are always going to do something with what is given to you 
- `read` somewhat implies that you want to see the state of tracked value, but is ambigous about if you want to do anything with that information

`get`, in particular, (while ~ unfortunately ~, matches legacy naming in our history), matches existing JS concepts from Map, WeakMap, other other concepts.

### Naming: `set`

- `set` implies mutation and potential side-effectful behavior, which is exactly what is happening
- `write` implies that data is being _written_ somewhere, which may not be true, depending on your point of view. There may also not even be state within the tracked value (as in the case of the "side-signals" example)

### Extension

If folks wanted, they could make their own tracked value with previous or historical values. This could be useful for extremely expensive operations that depend on previous computations. 
To do this, folks would need to implement their own class:
```js
class TrackedValueWithHistory {
    #previous;
    #current = tracked();

    get current() {
        return this.#current.current;
    }
    set current(value) {
        this.#previous = this.#current.current;
        this.$current.current = value;
    }

    get previous() {
        return this.#previous;
    }

    set(value) {
        this.current = value; 
    }

    // ...
}
```
