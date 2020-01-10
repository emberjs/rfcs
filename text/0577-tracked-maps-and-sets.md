- Start Date: 2020-01-02
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/577
- Tracking: (leave this empty)

# Tracked Maps and Sets

## Summary

Adds tracked versions of JavaScript's built-in `Map`, `WeakMap`, `Set`, and
`WeakSet` classes, and updates the `{{get}}` helper to work with `Map` and
`WeakMap`.

```js
class Store {
  inventory = new TrackedMap([
    ['socks', 123],
    ['shoes', 456],
  ]);

  employees = new TrackedSet([
    'Ed',
    'Katie',
    'Leah',
  ]);
}
```

## Motivation

Autotracking has overall been a huge success in Ember Octane, and has received
a lot of good feedback in general. However, there are a few use cases that are
not supported very well currently. One of those use cases is tracking changes to
_dynamic collections of values_, where its not possible to decorate properties
with the `@tracked` decorator because the properties themselves are constantly
changing.

There are four types of primitive/built-in collections in JavaScript:

1. Plain-Old-JavaScript-Objects (POJOs), which can operate as simple key-value
   stores, but have many other unrelated use cases
2. Maps, which are explicit key-value stores (and somewhat more flexible than
   POJOs)
3. Arrays, for storing lists of items
4. Sets, which are effectively unique arrays, and are usually used specifically
   to define groups and check whether a value exists within said group

These collections are used frequently for a variety of purposes in standard JS,
and each was introduced to solve use cases that the others could not. In
particular, `WeakMap` and `WeakSet` are very valuable for storing private meta
information about an object while still allowing it to be garbage collected, and
this ability is used very commonly in library and framework code.

Based on early feedback, it has become apparent that providing tracked versions
of each of these collections is important. In particular, a tracked version of a
key-value store is something that many developers have been wanting. Meanwhile,
the [`tracked-built-ins`](https://github.com/pzuraq/tracked-built-ins) library
has shown that tracked versions of `WeakMap` and `WeakSet` can be built, and are
very useful for developing various tracking utilities (such as the
[`@localCopy` decorator](https://github.com/pzuraq/tracked-toolbox#localcopy)
provided by `tracked-toolbox`).

POJOs are, unfortunately, not possible to address with native `Proxy`, which
cannot be used until IE11 support has been dropped. Arrays are fairly complex on
their own, as so have been broken out into
[their own RFC](https://github.com/emberjs/rfcs/pull/569). This RFC focuses on
adding the remaining two collection types - Maps and Sets.

## Detailed design

Ember will provide 4 classes importable from `@glimmer/tracking`:

```js
import {
  TrackedMap,
  TrackedSet,
  TrackedWeakMap,
  TrackedWeakSet,
} from '@glimmer/tracking';
```

These classes will all implement the exact same APIs as their built-in
equivalents, and in general will be transparent wrappers around them, with
read-methods entangling with autotracking, and write-methods triggering updates.
Any new methods that are added to the native equivalents will be added to these
classes as soon as possible, without need for a followup RFC, _unless_ the new
methods would change the behaviors of the tracked versions significantly.

For instance, there currently is no overlap of read/write methods in any of
these classes, so there is no conceptual conflict between entangling and
invalidating on a given method. If a method was added that was used to both read
and write to a Map or Set, it would require an RFC to be added to the tracked
version in order to define the behavior it should have.

### `TrackedMap` and `TrackedWeakMap` Tracking Dynamics

The tracked versions of maps will entangle and invalidate on a per-key basis.

```js
class Store {
  inventory = new TrackedMap([
    ['socks', 123],
    ['shoes', 456],
  ]);

  get socksCount() {
    return this.inventory.get('socks');
  }

  get shoesCount() {
    return this.inventory.get('shoes');
  }

  updateSocks(newCount) {
    this.inventory.set('socks', newCount);
  }

  updateShoes(newCount) {
    this.inventory.set('shoes', newCount);
  }
}
```

In this example, the `updateSocks` method would invalidate and trigger an update
to the `socksCount` getter, but it would not trigger updates to the `shoesCount`
getter. This is because maps are often used for non-iteration-based use cases,
and in these cases it would be much more performant to invalidate only the
related values when a particular key is updated.

In addition, the `size` property will be updated accordingly as values are added
to and removed from the map, and an iteration tag will be entangled whenever the
entire collection is iterated, and invalidated whenever any change is made.

### `TrackedSet` and `TrackedWeakSet` Tracking Dynamics

The tracked versions of sets will entangle and invalidate membership of
individual items.

```js
class Store {
  employees = new TrackedSet([
    'Ed',
    'Katie',
    'Leah',
  ]);

  get hasEd() {
    return this.employees.has('Ed');
  }

  get hasYehuda() {
    return this.employees.has('Yehuda');
  }

  addYehuda() {
    this.employees.add('Yehuda');
  }
}
```

In this example, `hasYehuda` would be invalidated if we called the `addYehuda`
method, while `hasEd` would not.

In addition, the `size` property will be updated accordingly as values are added
to and removed from the set, and an iteration tag will be entangled whenever the
entire collection is iterated, and invalidated whenever any change is made.

### `instanceof`

The goal of these classes is to transparently wrap the functionality of their
built-in counterparts. As such, they will return `true` to `instanceof` checks
against the built-in versions.

```js
let trackedMap = new TrackedMap();

trackedMap instanceof Map; // true
```

This will allow them to be passed to external libraries and used transparently,
providing a way for autotracking to be used with non-Ember libraries.

### `{{get}}` Interop

As mentioned in the motivation section, part of the reason to introduce tracked
maps and sets now is to provide users with a reactive key-value store, since
POJOs cannot be used for this purpose without using `Ember.get` and `Ember.set`
and this is a fairly common use case. Maps are iterable with both `{{each}}` and
`{{each-in}}` in templates, and users can use the `Map#get` and `Map#set`
methods in JavaScript code, but this leaves one crucial gap in functionality:
Accessing specific keys within templates.

```js
class Store extends Component {
  products = {
    socks: 123,
    shoes: 456,
  };
}
```
```hbs
Socks in Stock: {{this.products.socks}}
Shoes in Stock: {{this.products.shoes}}
```

To cover this gap, the `{{get}}` helper's functionality will be extended to work
with both `Map` and `TrackedMap`.

```js
class Store extends Component {
  products = new TrackedMap([
    ['socks', 123],
    ['shoes', 456],
  ]);
}
```
```hbs
Socks in Stock: {{get this.products "socks"}}
Shoes in Stock: {{get this.products "shoes"}}
```

This will allow users to use maps directly in templates in cases where this
functionality is needed.

## How we teach this

`TrackedMap` and `TrackedSet` are based on two common JS built-ins, and
`TrackedMap` will be the only reactive key-value store by default. As such, they
should be mentioned in the component guides when reactivity is first brought up,
possibly in the same location as arrays/collections in general. They should also
be discussed in more detail in an in-depth guide on reactive collections, along
with `TrackedWeakMap` and `TrackedWeakSet`, which are much less commonly used.

### API Docs

TODO

## Drawbacks

- This RFC commits us to supporting transparent wrappers around maps and sets.
  This could be difficult to support in the future if functionality is ever
  added to the built-in versions that cannot be implemented in userland code.
  However, the interoperability benefits with the rest of the ecosystem are a
  huge plus, and given the fact that `Map` and `Set` (and their `Weak`
  counterparts) have been designed with much more rigor, in an era where TC39
  has valued consistency, it is much less likely that the language would add
  functionality that was fundamentally not available/wrappable in userland. In
  the worst case, it would most likely mean having to use native `Proxy` to wrap
  them instead.

## Alternatives

- Instead of making `{{get}}` work on maps, wait for native `Proxy` to become
  available, and focus on adding a tracked version of POJOs that can work with
  standard dot notation (and `{{get}}`).

- Ship autotracking primitives and allow these classes to be implemented within
  addons or libraries. This is effectively already the situation with
  `tracked-built-ins`, and it hasn't been long enough to know for certain
  whether or not this library will be used commonly enough for its usage alone
  to be justification to be included in Ember's core.

  This RFC is being made pre-emptively, on the basis that it fills a gap that is
  _fundamental_ and will eventually be required commonly in most apps and many
  addons, at least once.
