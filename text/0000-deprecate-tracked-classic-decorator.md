---
Stage: Accepted
Start Date: 2021-03-18
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR:
---

# Deprecate using tracked in classic classes

## Summary

Deprecates using `tracked` to create tracked properties in classic classes:

```js
const MyClass = EmberObject.extend({
  someProp: tracked({ value: 123 }),
});
```

## Motivation

Using `tracked` within classic classes was seen as an interop step that would
help to ease the transition from classic to native, at the same time as adopting
Octane. However, it has seen little usage so far. In the ecosystem, there is not
a single usage in any addon (according to Ember Observer), and in general it
seems to serve a very niche case.

Since this feature was implemented, we also realized that it may be very useful
to use the function call form of `tracked()` to represent something other than a
classic decorator. For instance, the [tracked-built-ins](https://github.com/pzuraq/tracked-built-ins)
library allows users to create tracked versions of arrays, objects, maps, and
sets with `tracked()`:

```js
import { tracked } from 'tracked-built-ins';

class Foo {
  @tracked value = 123;

  obj = tracked({});
  arr = tracked([]);
  map = tracked(Map);
  set = tracked(Set);
  weakMap = tracked(WeakMap);
  weakSet = tracked(WeakSet);
}
```

This is a nice short hand which means users do not need to import classes like
`TrackedObject`, `TrackedArray`, and so on for a reasonably frequent use case.

Deprecating the usage of `tracked()` in classic classes would essentially allow
us to reclaim its usage, and gives us the possibility of adding something like
this API in the next major version of Ember. It also means we could do other
things with the function call version of it, such as add additional args to pass
to the decorator form.

As a final note, this will also set us up to more cleanly separate out the
`@glimmer/tracking` package from Ember, so that it can be built as a normal JS
package using standard build tools and Embroider.

## Transition Path

### Deprecation Guide

Using `tracked` as a classic decorator within classic classes has been
deprecated. In order to use tracked properties, you must convert to using
a native class.

Before:

```js
const MyClass = EmberObject.extend({
  someProp: tracked({ value: 123 }),
});
```

After:

```js
class MyClass {
  @tracked someProp = 123;
}
```

Alternatively, you can return to using classic idioms such as `get` and `set`
to get and set the property:

```js
const MyClass = EmberObject.extend({
  someProp: 123,

  updateProp() {
    this.set('someProp', 456);
  }
});
```

## How We Teach This

See deprecation guide.

## Drawbacks

- Introduces some churn
- Ties using tracked properties to native classes. Users will not have a choice
  to use tracked properties with classic class syntax anymore.

## Alternatives

- Export a new value, `trackedClassic`, from `@ember/object/compat`. This would
  _only_ be usable as a classic decorator in classic classes, and the current
  form of `tracked()` would still be deprecated.

- Use a Proxy to allow the result of `tracked()` to be used as both a classic
  decorator and whatever other value we decide it should return. This seems like
  it ties concerns, and it does not allow us to separate the `@glimmer/tracking`
  package from Ember.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
