---
Start Date: 2015-09-11
RFC PR: https://github.com/emberjs/rfcs/pull/91
Ember Issue: #12224 / #12990 / #13688

---

# Summary

Introduce `Ember.WeakMap` (`@ember/weakmap`), an ES6 enspired WeakMap. A
WeakMap provides a mechanism for storing and retriving private state. The
WeakMap itself does not retain a reference to the state, allowing the state to
be reclaimed when the key is reclaimed.

A traditional WeakMap (and the one that will be part of the language) allows
for weakness from key -> map, and also from map -> key. This allows either the
Map, or the key being reclaimed to also release the state.

Unforunately, this bi-directional weakness is problemative to polyfil. Luckily,
uni-directional weakness, in either direction, "just works". A polyfil must
just choose a direction.

*Note: Just like ES2015 WeakMap, only non null Objects can be used as keys*
*Note: `Ember.WeakMap` can be used interchangibly with the ES2015 WeakMap. This
will allow us to eventually cut over entirely to the Native WeakMap.*
 
# Motivation

It is a common pattern to want to store private state about a specific object.
When one stores this private state off-object, it can be tricky to understand
when to release the state. When one stores this state on-object, it will be
released when the object is released. Unfortunately, storing the state
on-object without poluting the object itself is non-obvious.

As it turns out, Ember's Meta already solves this problem for
listeners/caches/chains/descriptors etc. Unfortunately today, there is no
public API for apps or addons to utilize this. `Ember.WeakMap` aims to be
exactly that API.

Some examples:

* https://github.com/offirgolan/ember-cp-validations/blob/master/addon/utils/cycle-breaker.js
* https://github.com/stefanpenner/ember-state-services/ (will soon utilize the user-land polyfil of this) to prevent common leaks.

# Detailed design

## Public API

```js
import WeakMap from '@ember/weak-map'

var private = new WeakMap();
var object = {};
var otherObject = {};

private.set(object, {
  id: 1,
  name: 'My File',
  progress: 0
}) === private;

private.get(object) === {
  id: 1,
  name: 'My File',
  progress: 0
});


private.has(object) === true;
private.has(otherObject) === false;

private.delete(object) === private;
private.has(object) === false;
```

## Implementation Details

The backing store for `Ember.WeakMap` will reside in a lazy `ownMap` named
`weak` on the key objects `__meta__` object.

Each `WeakMap` has its own internal GUID, which will be the name of its slot,
in the key objects meta weak bucket. This will allow one object to belong in
multiple weakMaps without chance of collision.

Concrete Implementation: https://github.com/emberjs/ember.js/pull/12224
Polyfill: https://www.npmjs.com/package/ember-weakmap

# Drawbacks

* implementing bi-direction Weakness in userland is problematic.
* Using WeakMap will insert a non-enumerable `meta` onto the key Object.

# Alternatives

* Weakness could be implemented in the other direction, but this has questionable utility.

# Unresolved questions

N/A
