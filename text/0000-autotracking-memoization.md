- Start Date: 2020-04-17
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Autotracking Memoization

## Summary

Provides a low-level primitive for memoizing the result of a function based on
autotracking, allowing users to create their own reactive systems that can
respond to changes in autotracked state

```js
import { tracked } from '@glimmer/tracking';
import { memoizeTracked } from '@glimmer/tracking/primitives';

let count = 0;

class Person {
  @tracked firstName = 'Jen';
  @tracked lastName = 'Weber';

  fullName = memoizeTracked(() => {
    count++;
    return `${this.firstName} ${this.lastName}`;
  })
}

let person = new Person();

console.log(person.fullName()); // Jen Weber
console.log(count); // 1;
console.log(person.fullName()); // Jen Weber
console.log(count); // 1;

person.firstName = 'Jennifer';

console.log(person.fullName()); // Jennifer Weber
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
not be possible to build this decorator with public APIs.

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

This RFC proposes two functions to be added to Ember's public API:

```ts
function memoizeTracked<T, Args>(fn: (...args: Args) => T): (...args: Args) => T;

function isConst(fn: () => unknown): boolean;
```

These functions are exposed as exports of the `@glimmer/tracking/primitives`
module:

```ts
import {
  memoizeTracked,
  isConst
} from '@glimmer/tracking/primitives';
```

### Usage

`memoizeTracked` receives a function, and returns the same function, memoized.
The function will only rerun when the tracked values that have been _consumed_
while it was running have been _dirtied_. Otherwise, it will return the
previously computed value.

```ts
class State {
  @tracked value;
}

let state = new State();
let count = 0;

let counter = memoizeTracked(() => {
  // consume the state
  state.value;

  return count++;
});

counter(); // 1
counter(); // 1

state.value = 'foo';

counter(); // 2
```

Memoized functions are wrapped transparently, so they still accept the same
arguments and return the same value as the original function. This gives
`memoizeTracked` more flexibility in general and helps to clean up common usage
patterns, such as memoizing a method on a class.

```js
import { tracked } from '@glimmer/tracking';
import { memoizeTracked } from '@glimmer/tracking/primitives';

class Person {
  @tracked firstName = 'Jen';
  @tracked lastName = 'Weber';

  fullName = memoizeTracked(() => {
    return `${this.firstName} ${this.lastName}`;
  })
}
```

The results of memoized functions are _also_ consumed, so they can be nested.

```ts
let inner = memoizeTracked(() => { /* ... */ })

let outer = memoizeTracked(() => {
  /* ... */

  inner();
});
```

In this example, the outer function will be invalidated whenever the inner
function invalidates. This can be used to optimize a tracked function.

Arguments passed to the function are _not_ memoized. `memoizeTracked` only
memoizes based on autotracking, so it may or may not rerun if the same set of
arguments is passed to the function, or if a different set is passed.

#### Function Name

The `memoizeTracked` name was chose for two main reasons:

1. It's a fairly verbose name, which discourages common usage. It is meant to be
   a low-level API, and shouldn't make its way into common usage in app code.
2. The function name begins with `memo` instead of `track`, which means that
   auto-import completion won't show it when users type `@tracked`. This will
   make it less likely for users to stumble upon it without any context.

### Constant Functions

Memoized functions will only recompute if any of the tracked inputs that were
consumed previously change. If there _were_ no consumed tracked inputs, then
they will never recompute.

```ts
let count = 0;

let counter = memoizeTracked(() => {
  return count++;
});

counter(); // 1
counter(); // 1
counter(); // 1

// ...
```

When this happens, it often means that optimizations can be made in the code
surrounding the computation. For instance, in the Glimmer VM, we don't emit
updating bytecodes if we detect that a memoized function is constant, because
it means that this piece of DOM will never update.

In order to check if a memoized function is constant or not, users can use the
`isConst` function:

```ts
import { memoizeTracked, isConst } from '@glimmer/tracking/primitives';

class State {
  @tracked value;
}

let state = new State();
let count = 0;

let counter = memoizeTracked(() => {
  // consume the state
  state.value;

  return count++;
});


let constCounter = memoizeTracked(() => {
  return count++;
});

counter();
constCounter();

isConst(counter); // false
isConst(constCounter); // true
```

It is not possible to know whether or not a function is constant before its
first usage, so `isConst` will throw an error if the function has never been
called before.

```ts

let constCounter = memoizeTracked(() => {
  return count++;
});

isConst(constCounter); // throws an error, `constCounter` has not been used
```

This helps users avoid missing optimization opportunities by mistake, since most
optimizations happen on the first run only. If a user calls `isConst` on the
function prior to the first run, they may assume that the function is
non-constant on accident.

If called on a normal, non-memoized function, `isConst` will always return
`false`. This gives the user some flexibility in how they structure their code,
allowing them to memoize some functions but not others and still optimize them
with `isConst`.

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

#### `memoizeTracked`

Receives a function, and returns a wrapped version of it that memoizes based on
_autotracking_. The function will only rerun whenever any tracked values used
within it have changed. Otherwise, it will return the previous value.

```ts
import { tracked } from '@glimmer/tracking';
import { memoizeTracked } from '@glimmer/tracking/primitives';

class State {
  @tracked value;
}

let state = new State();
let count = 0;

let counter = memoizeTracked(() => {
  // consume the state. Now, `counter` will
  // only rerun if `state.value` changes.
  state.value;

  return count++;
});

counter(); // 1

// returns the same value because no tracked state has changed
counter(); // 1

state.value = 'foo';

// reruns because a tracked value used in the function has changed,
// incermenting the counter
counter(); // 2
```

#### `isConst`

Can be used to check if a memoized function is _constant_. If no tracked state
was used while running a memoized function, it will never rerun, because nothing
can invalidate its result. `isConst` can be used to determine if a memoized
function is constant or not, in order to optimize code surrounding that
function.

```ts
import { tracked } from '@glimmer/tracking';
import { memoizeTracked, isConst } from '@glimmer/tracking/primitives';

class State {
  @tracked value;
}

let state = new State();
let count = 0;

let counter = memoizeTracked(() => {
  // consume the state
  state.value;

  return count++;
});


let constCounter = memoizeTracked(() => {
  return count++;
});

counter();
constCounter();

isConst(counter); // false
isConst(constCounter); // true
```

If called on a function that is _not_ memoized, `isConst` will always return
`false`, since that function will always rerun.

If called on a memoized function that hasn't been run yet, it will throw an
error. This is because there's no way to know if the function will be constant
or not yet, and so this helps prevent missing an optimization opportunity on
accident.

## Drawbacks

- The usage of the term `memoize` may mislead users. It is a _correct_ usage, as
  in it meets the definition of memoization:

  > In computing, memoization or memoisation is an optimization technique used primarily to speed up computer programs by storing the results of expensive function calls and returning the cached result when the same inputs occur again.

  The main difference being that most implementations of memoization that users
  are familiar with consider the inputs to be the _arguments_ to the function.
  Autotracking takes a somewhat novel approach here by saying it memoizes based
  on the inputs used _during_ the calculation.

  This terminology difference is teachable, and given this is a low-level API
  which is not expected to be used commonly by average users, it makes more
  sense than alternatives like `autotrackedFunction` since it describes what the
  function becomes - memoized.

## Alternatives

- Stick with higher level APIs and don't expose the primitives. This could lead
  to an explosion of high level complexity, as Ember tries to provide every type
  of construct for users to use, rather than

- `memoizeTracked` could return an object with a `value()` function instead of
  the function itself. This would give a more natural place for setting metadata
  such as `isConst`, but would sacrifice some of the ergonomics of the API. It
  also would require more object creations, and given this is a very commonly
  used, low-level API, it would make sense for it to avoid a design that could
  be limited in terms of performance in this way.

- `trackedMemoize` is a somewhat more accurate name that may make more sense to
  the average user. It does however collide with the `tracked` import, and will
  likely pop up in autocomplete tooling because of this, which would not be
  ideal. As a low-level API, this function should generally not be very visible
  to average users.
