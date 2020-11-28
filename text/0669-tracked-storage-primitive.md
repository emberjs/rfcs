- Start Date: 2020-09-26
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Tracked Storage Primitives

## Summary

Provide a low level primitive for storing tracked values:

```js
let storage = createStorage();

// Set the value of the storage
setValue(storage, 123);

// Get the value of the storage
getValue(storage);
```

## Motivation

Today, there are two methods for creating a tracked value in Ember apps:

1. The `@tracked` decorator, which turns class properties into tracked
   properties
2. `Ember.get`, which can be used to track a key on an object so that it updates
   if it is set with `Ember.set`, or dirtied with `notifyPropertyChange`.

The `@tracked` decorator is good for cases where there are a known set of
mutable values on a class, which is one of the most common use cases in Ember
applications. There are a number of cases, however, where there are unknown,
dynamic, or potentially infinite numbers of decoratable keys. An example of this
is _maps_.

[Maps](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map)
are used as key-value stores, typically in cases where the keys are not known
upfront or are dynamic. In many cases, there are many thousands of potential
keys that could be used in a map, even if only a few ever actually are used.
Decorating each and every potential key on a class in these types of cases is
not practical, and if non-alpha-numeric keys are used, not possible.

There are a few different ways that this type of use case can be solved with
public APIs today:

1. We can use the "immutable pattern", where the map is duplicated whenever we
   wish to change it, the change is applied, and the result is re-assigned to
   the tracked property the map is assigned to.

   ```js
   @tracked map = new Map();

   updateMapValue(key, value) {
     let newMap = new Map(this.map);
     newMap.set(key, value);
     this.map = newMap;
   }
   ```

   This pattern works nicely when followed rigorously, and was the initial
   pattern proposed for _all_ of these types of data structures and operations
   (it's even in the official documentation). However, it is fairly unergonomic
   without the aid of additional libraries, such as [Immer.js](https://github.com/immerjs/immer),
   and it incurs a decent amount of overhead, especially in cases where the data
   structures are large.

   One solution for that is to re-set the tracked property with the same value,
   after the mutation:

   ```js
   @tracked map = new Map();

   updateMapValue(key, value) {
     this.map.set(key, value);
     this.map = this.map;
   }
   ```

   This pattern is problematic however, because it's not obvious what the
   purpose of re-setting the property is. A developer unfamiliar with Ember may
   mistakenly believe this was an error, and remove this statement, causing
   bugs. It's not a very google-able pattern either, which makes it more
   difficult to learn and to teach. Finally, it's fairly easy this way for the
   state to get out of sync with the rest of the system if a developer forgets
   to add the extra re-set after mutating the value.

   Over time, it's become clear that this pattern is in actuality an antipattern
   for these reasons, and something we should begin to discourage in general.

   Note that the _original_ immutable pattern, in which state is fully copied,
   does not have these problems, and remains a fully valid pattern. However, due
   to performance constraints, ergonomics, and personal preferences, it does not
   make sense for _every_ Ember application, so we still need to enable
   alternative patterns.

2. We can continue to use `Ember.get` and `Ember.set` with plain objects.
   This pattern works now and will continue to work for the forseeable future,
   but does not support using non-string values as keys, so it cannot cover all
   the same use cases as Map. In addition, it's not practical for other types of
   collections, such as as arrays, since `get` and `set` were not built to work
   with them.

   In addition, it presents a mismatch between the models put forward in Octane,
   and the models of Ember classic. In Octane, the idea is that the details of
   state tracking are _opaque_ to the user. Using `get` and `set` manually means
   that every time the object is accessed or updated, the user has to
   consciously remember to use these special functions - they have to _think_
   about the details of tracking state. This is a bug-prone process, because it
   only takes one mistake in one usage, and the state itself could be used in
   many places. In Octane, users should only need to declare reactivity once -
   when the state itself is defined.

3. We can use the Cell pattern, as described in [this blog post](https://www.pzuraq.com/autotracking-case-study-trackedmap/).
   This pattern is robust, and can be used to make many types of tracked data
   structures, as the [tracked-built-ins](https://github.com/pzuraq/tracked-built-ins)
   library shows. There are a few main downsides to it:

   - Since it is not a formalized pattern, anyone who wants to use it has to
     reimplement it. This is not a huge amount of code to add to any given
     addon, but it means addon authors have to rederive the pattern themselves
     from scratch, which is not ideal. It means there is more overhead overall
     to learning these patterns in the first place.
   - It is not an optimal pattern, because it adds a layer of decoration and an
     instance of a class for every cell. This is not a major amount of overhead,
     but ideally we wouldn't incur it at all.
   - It has no potential for future extensibility, particularly with regards to
     _debug_ tooling. Ideally, tracked state, even custom tracked state, should
     be easy to understand and debug with standard Ember tooling. Currently,
     when users look at a stack trace with an issue caused by a cell, they get
     something that is very difficult to understand ("why is it telling me
     there's an issue with setting `value` when I'm setting a completely
     different key on my TrackedMap?")

Out of these patterns, the Cell pattern is the most robust and most in-line with
state management in Ember Octane. This RFC seeks to formalize it and add it to
the framework, so we can standardize on a common pattern and address the issues
that it currently has.

## Detailed design

This RFC proposes adding the following APIs, exported from
`@glimmer/tracking/primitives/storage`:

```ts
declare function createStorage<T>(
  initialValue?: T,
  isEqual?: (oldValue: T, newValue: T) => boolean
): Storage<T>;

declare function getValue<T>(storage: Storage<T>): T;

declare function setValue<T>(storage: Storage<T>, value: T): void;
```

### `createStorage`

This function creates and returns an instance of a tracked storage. Tracked
storage instances contain one value, which can be read with `getValue` and set
with `setValue`. Reading the value consumes it, so that any tracked computations
which are currently active will become entangled with it, and setting value
later will dirty it and cause them to recompute. Creating the value does _not_
consume it.

`createStorage` can receive an optional initial value as its first parameter.
It also receives an optional `isEqual` function. This function runs whenever the
value is about to be set, and determines if the value is equal to the previous
value. If it is equal, it does not set the value or dirty it. This defaults to
`===` equality.

### `getValue`

This function receives a tracked storage instance, and returns the value it
contains, consuming it so that it entangles with any currently active
autotracking contexts.

### `setValue`

This function receives a tracked storage instance and a value, and sets the
value of the tracked storage to the passed value if it is not equal to the
previous value. If the value is set, it will dirty the storage, causing any
tracked computations which consumed the stored value to recompute. If the value
was _not_ changed, then it does not set the value or dirty it.

### Re-implementing `@tracked` with storage

We can see how the `@tracked` decorator itself can be re-implemented using
tracked storage.

```js
function tracked(target, key, { initializer }) {
  let storages = new WeakMap();

  function getStorage(obj) {
    let storage = storages.get(obj);

    if (storage === undefined) {
      storage = createStorage(initializer.call(this), () => true);
      storages.set(this, storage);
    }

    return storage;
  };

  return {
    get() {
      return getValue(getStorage(this));
    },

    set(value) {
      return setValue(getStorage(this), value);
    },
  };
}
```

### A note on this design

In general, API design and cohesiveness is an important value in Ember. We work
very hard as a community to develop patterns which are uniform across many
different types of APIs, which in turn leads to a better DX as developers can
generally know what to expect from a given API without even needing to read the
documentation.

The API proposed in this RFC, which uses functions to access and manipulate a
value via an opaque handle, is an unusual API design when compared to most other
Ember APIs. It is a pattern which provides extra flexibility to the implementers
of the API, at the cost of a _severe_ DX penalty for the users of the API,
because of this unusual-ness.

As such this pattern should be used extremely sparingly, on APIs which meet the
following criteria:

1. They are not expected to be used by _every_ Ember developer. If the API will
   be used in the Super Rentals tutorial, or the standard non-advanced sections
   of the guides, then it should not use this pattern.
2. They are for extremely core parts of Ember, such as autotracking, where the
   primitives underlying them could potentially change significantly in the
   future, and maximum flexibility is desired because of this.
3. They are for extremely performance critical parts of Ember, with usages in
   the hundreds of thousands or millions per app, and so maximum flexbility is
   desired in order to be able to leverage changes to the primitives for
   optimizing performance.

In general, most Ember APIs will _not_ meet these requirements, and so this API
design should not be used for them.

## How we teach this

The API docs for this feature can be taken from the descriptions of the
functions in the detailed design section of the RFC. The following guide should
be added to the Autotracking In-Depth guide in the official guides, and the
section on [POJOs](https://guides.emberjs.com/release/in-depth-topics/autotracking-in-depth/#toc_plain-old-javascript-objects-pojos)
should be removed.

### Creating custom tracked data structures

Tracked properties are a great solution for many different use cases. However,
sometimes they aren't the right tool for the job. For instance, you may need
have a problem that requires you to work with a number of dynamically created
properties, rather than ones that you know up front.

For example, we could make a scoreboard component that keeps score for an
arbitrary number of players, and keep track of the score for each player using
a JavaScript [Map](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Map).

```js
// app/components/scoreboard.js
import Component from '@glimmer/component';
import { action } from '@ember/object';

export default class Scoreboard extends Component {
  // Mock data for players - this would eventually be replaced with an argument
  // so players could be passed in
  players = [
    { name: 'Zoey' },
    { name: 'Tomster' },
  ];

  // Create a map from player -> score, with each player's score starting at 0
  scores = new Map(this.players.map(p => [p, 0]));

  @action
  incrementScore(player) {
    let currentScore = this.scores.get(player);

    this.scores.set(player, currentScore + 1);
  }
}
```
```hbs
{{#each-in this.scores |player score|}}
  <div>
    {{player.name}}: {{score}}

    <button type="button" {{on "click" (fn this.incrementScore player)}}>
      +
    </button>
  </div>
{{/each-in}}
```

This component will loop over the map of scores and show each player's name and
score, along with a button to increment the score. The logic for this component
is all correct, but it won't update when we increment the score because the
`scores` map is not tracked state. Let's fix that!

Ember provides a set of primitive APIs for creating _tracked storage_. These
APIs can be used to create higher level tracked data structures. In fact, the
`@tracked` decorator is _implemented_ using tracked storage, and we can see how
it boils down to tracked storage under the hood in this example.

```js
import { tracked } from '@glimmer/tracking';

class Person {
  @tracked name;
}

// could be thought of as:
import {
  createStorage,
  getValue,
  setValue
} from '@glimmer/tracking/primitives/storage';

class Person {
  #name = createStorage();

  get name() {
    return getValue(this.#name);
  }

  set name(value) {
    setValue(this.#name, value);
  }
}
```

In this case, we'll create `TrackedMap`, a class that has the same public API
and behavior as `Map`, but is integrated with autotracking so that if we read or
update a value in the map our app will correctly rerender.

First off, let's start off with the basics for our class. We want it to have the
same semantics as a standard map, and there's no need to re-implement the wheel.
We can use a standard map as the backing storage for our custom wrapper, and
defer to it directly.

```js
// app/utils/tracked-map.js
class TrackedMap {
  #map = new Map();

  constructor(initialValues) {
    for (let [key, value] of initialValues) {
      this.set(key, value);
    }
  }

  get(key) {
    return this.#map.get(key);
  }

  set(key, value) {
    this.#map.set(key, value);
  }

  forEach(fn) {
    this.#map.forEach(fn);
  }

  // Other methods and APIs omitted...
}
```

Now, we need to add our tracked storage. We'll use the `createStorage` API to do
this.

```js
// app/utils/tracked-map.js
import {
  createStorage,
  getValue,
  setValue,
} from '@glimmer/tracking/primitives/storage';

class TrackedMap {
  #map = new Map();

  constructor(initialValues) {
    for (let [key, value] of initialValues) {
      this.set(key, value);
    }
  }

  _getStorage(key) {
    let storage = this.#map.get(key);

    if (storage === undefined) {
      storage = createStorage();
      this.#map.set(key, storage);
    }

    return storage;
  }

  get(key) {
    return getValue(this._getStorage(key));
  }

  set(key, value) {
    setValue(this._getStorage(key), value);
  }

  forEach(fn) {
    this.#map.forEach((storage, key) => fn(getValue(storage, key, this)));
  }

  // Other methods and APIs omitted...
}
```

Now, instead of setting values directly in our internal private map, we're
wrapping those values in tracked storage. We create the storage instances with
`createStorage`, we get the value from them with `getValue`, and we set
the value in the with `setValue`. Each tracked storage contains exactly one
value, and whenever we read that value with `getValue`, it is consumed in any
tracked computations that are currently occuring - just like reading the value
of a tracked property. Likewise, setting the value of a tracked storage instance
with `setValue` will dirty it, and cause any tracked computations which used it
previously to recompute, just like setting the value of a tracked property.

There's just one issue with this implementation so far - what happens to
`forEach` if we add a _new_ value to the `TrackedMap`, one that didn't exist
before? Currently, `forEach` will consume every _existing_ value and entangle
it, but it will not rerun if we were to add a new key to the map.

Logically, the thing we're trying to represent here is the state of the
collection's iteration. We can do this by creating another storage, and placing
the collection in it.

```js
// app/utils/tracked-map.js
import {
  createStorage,
  getValue,
  setValue,
} from '@glimmer/tracking/primitives/storage';

class TrackedMap {
  #map = new Map();
  #collection = createStorage(this, () => true);

  constructor(initialValues) {
    for (let [key, value] of initialValues) {
      this.set(key, value);
    }
  }

  _getStorage(key) {
    let storage = this.#map.get(key);

    if (storage === undefined) {
      storage = createStorage();
      this.#map.set(key, storage);
    }

    return storage;
  }

  get(key) {
    return getValue(this._getStorage(key));
  }

  set(key, value) {
    // dirty the collection state
    setValue(this.#collection, this);
    setValue(this._getStorage(key), value);
  }

  forEach(fn) {
    // Entangle the collection state
    getValue(this.#collection);
    this.#map.forEach((storage, key) => fn(getValue(storage, key, this)));
  }

  // Other methods and APIs omitted...
}
```

Now we have a storage that we use to represent the value of the collection
itself. We pass a custom equality function to this storage which always returns
`true`, meaning that whenever we set the value of this storage, it will _always_
notify, even if the value hasn't changed. We then entangle the collection
storage in `forEach` by using `getValue` on it, even though we don't actually
use the value, and we dirty with `setValue` whenever we `set` a value. Since we
used a custom equality function that will always notify, we can set it to the
collection itself again.

That should cover all of the basic functionality! If we now use `TrackedMap` in
our original example instead of `Map` everything should update and work as
expected:

```js
// app/components/scoreboard.js
import Component from '@glimmer/component';
import { action } from '@ember/object';
import TrackedMap from '../utils/tracked-map';

export default class Scoreboard extends Component {
  // Mock data for players - this would eventually be replaced with an argument
  // so players could be passed in
  players = [
    { name: 'Zoey' },
    { name: 'Tomster' },
  ];

  // Create a map from player -> score, with each player's score starting at 0
  scores = new TrackedMap(this.players.map(p => [p, 0]));

  @action
  incrementScore(player) {
    let currentScore = this.scores.get(player);

    this.scores.set(player, currentScore + 1);
  }
}
```
```hbs
{{#each-in this.scores |player score|}}
  <div>
    {{player.name}}: {{score}}

    <button type="button" {{on "click" (fn this.incrementScore player)}}>
      +
    </button>
  </div>
{{/each-in}}
```

## Alternatives

- Stick with higher level APIs and don't expose the primitives. This could lead
  to an explosion of high level complexity, as Ember tries to provide every type
  of construct for users to use, rather than a low level primitive.

- Expose a more functional or more object-oriented API. This would be a somewhat
  higher level API than the one proposed here, which may be a bit more
  ergonomic, but also would be less flexible. Since this is a new primitive and
  we aren't sure what features it may need in the future, the current design
  keeps the implementation open and lets us experiment without foreclosing on a
  possible higher level design in the future. It also matches with the
  previously added primitive APIs.

## Open Questions

- Should we add a `peekValue` API to check the value without entangling it? This
  API is something we've avoided doing for some time because it's very easy to
  accidentally cause bugs with it, but it's also something that is possible to
  emulate today with a little extra bookkeeping. It's something that has been
  asked for a few times as well.

- Should we add some debug tooling information as the final argument to these
  APIs? If so, what should that information be?
