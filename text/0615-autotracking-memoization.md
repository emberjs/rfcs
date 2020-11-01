---
Start Date: 2020-04-17
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/615
Tracking: (leave this empty)

---

# Autotracking Memoization

## Summary

Provides a low-level primitive for memoizing the result of a function based on
autotracking, allowing users to create their own reactive systems that can
respond to changes in autotracked state.

```js
import { tracked } from '@glimmer/tracking';
import { createCache, getValue } from '@glimmer/tracking/primitives/cache';

let computeCount = 0;

class Person {
  @tracked firstName = 'Jen';
  @tracked lastName = 'Weber';

  #fullName = createCache(() => {
    ++computeCount;
    return `${this.firstName} ${this.lastName}`;
  })

  get fullName() {
    return getValue(this.#fullName);
  }
}

let person = new Person();

console.log(person.fullName); // Jen Weber
console.log(count); // 1;
console.log(person.fullName); // Jen Weber
console.log(count); // 1;

person.firstName = 'Jennifer';

console.log(person.fullName); // Jennifer Weber
console.log(count); // 2;
```

## Motivation

Autotracking is the fundamental reactivity model within Ember Octane, and has
been highly successful so far in its usage and adoption. However, users today
can only integrate with autotracking via the `@tracked` decorator, which allows
them to _create_ tracked root state. There is no way to write code that responds
to changes in that root state directly - the only way to do so is indirectly via
Ember's templating layer.

An example of where users might want to do this is the [`@cached`](https://github.com/emberjs/rfcs/pull/566)
decorator for getters. This decorator only reruns its code when the tracked
state it accessed during its previous computation changes. Currently, it would
be quite difficult and error prone to build this decorator with public APIs.

For a more involved example, we can take a look at [ember-concurrency](http://ember-concurrency.com/),
which allows users to define async tasks. Ember Concurrency has an `observes()`
API for tasks, which allows tasks to rerun when a property changes. This API is
not documented or encouraged, and instead lifecycle hooks are recommended to
rerun tasks. However, in Octane, lifecycle hooks on components are no longer
available, removing that as an option. A more ergonomic, autotracked version of
concurrency tasks could be created if users had a way to react to changes in
autotracked state.

Data layers like Ember Data could also benefit from this capability. These
layers tend to have to keep state in sync between multiple levels of caching,
which traditionally was done with computed properties and eventing systems. The
ability to use autotracking to replace these systems, and to define their own
reactive semantics, could help complex libraries and data layers out immensely.

## Detailed design

This RFC proposes four functions to be added to Ember's public API:

```ts
interface Cache<T = unknown> {}

function createCache<T>(fn: () => T): Cache<T>;

function getValue<T>(cache: Cache<T>): T

function isConst(cache: Cache): boolean;
```

These functions are exposed as exports of the `@glimmer/tracking/primitives/cache`
module:

```ts
import {
  createCache,
  getValue,
  isCache,
  isConst,
} from '@glimmer/tracking/primitives/cache';
```

### Usage

`createCache` receives a function, and returns a cache instance for that function.
Users can call `getValue()` with the cache instance as an argument to run the
function and get the value of its output. The cache will then return the same
value whenever `getValue` is called again, until one of the tracked values that
was _consumed_ while it was running previously has been _dirtied_.

```ts
class State {
  @tracked value;
}

let state = new State();
let computeCount = 0;

let counter = createCache(() => {
  // consume the state
  state.value;

  return ++computeCount;
});

getValue(counter); // 1
getValue(counter); // 1

state.value = 'foo';

getValue(counter); // 2
```

Getting the value of a cache also _consumes_ the cache. This means caches can be
nested, and whenever you use a cache inside of another cache, the outer cache
will dirty if the inner cache dirties.

```ts
let inner = createCache(() => { /* ... */ })

let outer = createCache(() => {
  /* ... */

  inner();
});
```

This can be used to break up different parts of a execution so that only the
pieces that changed are rerun.

### Constant Caches

Caches will only recompute if any of the tracked inputs that were consumed
previously change. If there _were_ no consumed tracked inputs, then they will
never recompute.

```ts
let computeCount = 0;

let counter = createCache(() => {
  return ++computeCount;
});

getValue(counter); // 1
getValue(counter); // 1
getValue(counter); // 1

// ...
```

When this happens, it often means that optimizations can be made in the code
surrounding the computation. For instance, in the Glimmer VM, we don't emit
updating bytecodes if we detect that a memoized function can never change, because
it means that this piece of DOM will never update.

In order to check if a memoized function is constant or not, users can use the
`isConst` function:

```ts
import { createCache, getValue, isConst } from '@glimmer/tracking/primitives/cache';

class State {
  @tracked value;
}

let state = new State();
let computeCount = 0;

let counter = createCache(() => {
  // consume the state
  state.value;

  return ++computeCount;
});


let constCounter = createCache(() => {
  return ++computeCount;
});

getValue(counter);
getValue(constCounter);

isConst(counter); // false
isConst(constCounter); // true
```

It is not possible to know whether or not a cache is constant before its
first usage, so `isConst` will throw an error if the cache has never been
accessed before.

```ts

let constCounter = createCache(() => {
  return count++;
});

isConst(constCounter); // throws an error, `constCounter` has not been used
```

This helps users avoid missing optimization opportunities by mistake, since most
optimizations happen on the first run only. If a user calls `isConst` on the
function prior to the first run, they may assume that the function is
non-constant on accident.

## How we teach this

This topic is one that is meant for advanced users and library authors. It
should be covered in detail in the Autotracking In-Depth guide in the Ember
guides.

This guide should cover how memoization works, and various techniques for using
memoization. It should cover a variety of ways to use memoization to accomplish
common tasks of other reactivity systems. Pull-based reactivity is unfamiliar to
many programmers, so we should try to familiarize them with as many common
examples as possible.

Some possibilities include:

- Building the `@cached` decorator from scratch.
- Building a data layer that syncs changes to models to localStorage or a
  backend in real time, as the changes occur (note: requires polling of some
  kind, or a component to do this).
- Building a `RemoteData` implementation, a helpful wrapper that sends a fetch
  request to a remote url and loads data whenever the url input changes.

### API Docs

#### `createCache`

Receives a function, and returns a wrapped version of it that memoizes based on
_autotracking_. The function will only rerun whenever any tracked values used
within it have changed. Otherwise, it will return the previous value.

```ts
import { tracked } from '@glimmer/tracking';
import { createCache, getValue } from '@glimmer/tracking/primitives/cache';

class State {
  @tracked value;
}

let state = new State();
let computeCount = 0;

let counter = createCache(() => {
  // consume the state. Now, `counter` will
  // only rerun if `state.value` changes.
  state.value;

  return ++computeCount;
});

getValue(counter); // 1

// returns the same value because no tracked state has changed
getValue(counter); // 1

state.value = 'foo';

// reruns because a tracked value used in the function has changed,
// incermenting the counter
getValue(counter); // 2
```

#### `getValue`

Gets the value of a cache created with `createCache`.

```ts
import { tracked } from '@glimmer/tracking';
import { createCache, getValue } from '@glimmer/tracking/primitives/cache';

let computeCount = 0;

let counter = createCache(() => {
  return ++computeCount;
});

getValue(counter); // 1
```

#### `isConst`

Can be used to check if a memoized function is _constant_. If no tracked state
was used while running a memoized function, it will never rerun, because nothing
can invalidate its result. `isConst` can be used to determine if a memoized
function is constant or not, in order to optimize code surrounding that
function.

```ts
import { tracked } from '@glimmer/tracking';
import { createCache, getValue, isConst } from '@glimmer/tracking/primitives/cache';

class State {
  @tracked value;
}

let state = new State();
let computeCount = 0;

let counter = createCache(() => {
  // consume the state
  state.value;

  return computeCount++;
});


let constCounter = createCache(() => {
  return computeCount++;
});

getValue(counter);
getValue(constCounter);

isConst(counter); // false
isConst(constCounter); // true
```

If called on a cache that hasn't been accessed yet, it will throw an
error. This is because there's no way to know if the function will be constant
or not yet, and so this helps prevent missing an optimization opportunity on
accident.

## Alternatives

- Stick with higher level APIs and don't expose the primitives. This could lead
  to an explosion of high level complexity, as Ember tries to provide every type
  of construct for users to use, rather than a low level primitive.

- Expose a more functional or more object-oriented API. This would be a somewhat
  higher level API than the one proposed here, which may be a bit more
  ergonomic, but also would be less flexible. Since this is a new primitive and
  we aren't sure what features it may need in the future, the current design
  keeps the implementation open and lets us experiment without foreclosing on a
  possible higher level design in the future.
