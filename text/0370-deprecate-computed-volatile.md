---
Start Date: 2018-08-31
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/370
Tracking: https://github.com/emberjs/rfc-tracking/issues/17

---

# Summary

Deprecate `computed().volatile()` in favor of undecorated native getters and
setters.

# Motivation

`computed().volatile()` is a commonly misunderstood API. On its surface,
declaring a computed as volatile causes the computed to recalculate every time
it is called. This actually works much like native, undecorated accessors do on
classes, with one key difference.

Volatile properties are meant to respresent fundamentally unobservable values.
This means that they swallow notification changes, and will not notify under any
circumstances, and that when setting a volatile value the user must notify
manually:

```js
const Foo = EmberObject.extend({
  bar: computed({
    get() {
      return this._value;
    }

    set(key, value) {
      return this._value = value;
    }
  }).volatile(),

  baz: computed('bar', {
    get() {
      return this.bar;
    }
  }),
});

let foo = Foo.create();

foo.set('bar', 123);
foo.baz; // 123, it's the initial get so nothing cached yet

foo.set('bar', 456);
foo.baz; // 123, no property changes were made so the cache was not cleared
```

This behavior is useful at times for framework code, but is generally not what
users are expecting. By constrast, when using native accessors with `set` and
`get`, Ember treats them just like any other property. From its perspective,
they _are_ standard properties, so it'll continue to notify as expected.

```js
class Foo {
  get bar() {
    return this._value;
  }

  set bar(value) {
    this._value = value;
  }

  @computed('bar')
  get baz() {
    return this.bar;
  }
});

let foo = new Foo();

set(foo, 'bar', 123);
foo.baz; // 123, it's the initial get so nothing cached yet

set(foo, 'bar', 456);
foo.baz; // 456, cache was cleared and value was updated
```

The most common use case for volatile computeds was when users wanted a computed
to behave like a native getter/setter. Now that we (almost) _have_ those in a
easy to use form, it makes more sense to deprecate the volatile API and rely
directly on native functionality.

# Transition Path

Native getters and setters will _only_ work on native classes, due to how the
internals of the old object model work. To ensure that users do not accidentally
try to replace volatile with getters/setters on non-native classes, we should
provide 2 deprecation warnings:

1. Deprecation when users use volatile on a computed which tells them that the
  API has been deprecated, and that they'll need to update native class syntax
  to remove the volatile property.

2. Deprecation when users use volatile on a computed decorator (to be RFC'd)
  which tells them to remove the computed decorator entirely from the getter.

Volatile properties will be removed once native classes are the default.

# How We Teach This

In general documentation should be updated to use native getters and setters
wherever `volatile` was used. This will have to happen after docs are updated to
use native classes, because native getters and setters do _not_ work with the
older object model.

# Drawbacks

Volatility is useful for framework level concerns, for instance if developing an
API or decorator that already handles notification. Addon authors may be able to
use this functionality.

Not having an alternative for old style classes or mixins could be problematic
for users who aren't ready to update to native class syntax.

# Alternatives

We could keep `volatile()` around for any potential addons that may want to use
it, but teach native getters/setters as the preferred path for most use cases.

We could provide `volatile` as a separate API/decorator to distinguish it from
computed properties, and discourage use for users.
