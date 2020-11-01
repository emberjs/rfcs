---
Start Date: 2018-08-30
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/369
Tracking: https://github.com/emberjs/rfc-tracking/issues/18

---

# Summary

Deprecate computed overridability and `computed().readOnly()` in favor of
read-only computeds as the default.

# Motivation

Computed properties have existed in Ember long before class syntax and native
accessors (getters and setters) were readily available, and as such they have a
few notable behavioral differences. As we move toward adopting native class
syntax and using a decorator-based form of computeds, it makes sense to
reconcile these differences so that users can expect them to work the same as
their native counterparts.

The main and most notable difference this RFC seeks to deprecate is computed
overridability (colloquially known as "clobbering"). There are some other
notable differences, including the caching behavior of the `return` value of
setter functions, which may be addressed in future RFCs.

## Overridability

When defining a native getter without a setter, attempting to set the value will
throw a hard error (in strict mode):

```js
function makeFoo() {
  'use strict';

  class Foo {
    get bar() {
      return this._value;
    }
  }

  let foo = new Foo();

  foo.bar; // undefined
  foo.bar = 'baz'; // throws an error in strict mode
}
```

By constrast, computed properties without setters will be overridden when they
are set, meaning the computed property is removed from the object and replaced
with the set value:

```js
const Foo = EmberObject.extend({
  bar: computed('_value', {
    get() {
      return this._value;
    },
  }),
});

let foo = Foo.create();

foo.bar; // undefined
foo.set('bar', 'baz'); // Overwrites the getter
foo.bar; // 'baz'
foo.set('_value', 123);
foo.bar; // 'baz'
```

This behavior is confusing to newcomers, and oftentimes unexpected. Common best
practice is to opt-out of it by declaring the property as `readOnly`, which
prevents this overridability.

# Transition Path

This RFC proposes that `readOnly` properties become the default, and that in
order to override users must opt in by defining their own setters:

```js
class Foo {
  get bar() {
    if (this._bar) {
      return this._bar;
    }

    return this._value
  }

  set bar(value) {
    this._bar = value
  }
}
```

## Macros

Most computed macros are overridable by default, the exception being `readOnly`.
This RFC proposes that all computed macros with the exception of `reads` would
become read only by default. The purpose of `reads` is to _be_ overridable, so
its behavior would remain the same.

## Decorator Interop

It may be somewhat cumbersome to write overriding functionality or add proxy
properties when overriding is needed. In an ideal world, computed properties
would modify accessors transparently so that they could be composed with other
decorators, such as an `@overridable` decorator:

```js
class Foo {
  @overridable
  @computed('_value')
  get bar() {
    return this._value;
  }

  @overridable
  @and('baz', 'qux')
  quux;
}
```

Currently this is not possible as computed properties store their getter/setter
functions elsewhere and replace them with a proxy getter and the mandatory
setter assertion, respectively. In the long term, making computeds more
transparent in this way would be ideal, but it is out of scope for this RFC.

## Deprecation Timeline

This change will be a breaking change, which means we will not be able to change
the behavior of `computed` until Ember v4.0.0. Additionally, users will likely
want to continue using `.readOnly()` up until overriding has been fully removed
to ensure they are using properties safely. With that in mind, the ordering of
events should be:

1. Ember v3
    * Deprecate the default override-setter behavior immediately. This means that
      a deprecation warning will be thrown if a user attempts to set a
      non-`readOnly` property which does not have a setter. Users will still be
      able to declare a property is `readOnly` without a deprecation warning.
    * Add optional feature to change the deprecation to an assertion after the
      deprecation has been released, and to show a deprecation when using
      the `.readOnly()` modifier.
    * After the deprecation and optional feature have been available for a
      reasonable amount of time, enable the optional feature by default in new
      apps and addons. The main reason we want to delay this is to give _addons_
      a chance to address deprecations, since enabling this feature will affect
      both apps and the addons they consume.
2. Ember v4
    * Remove the override-setter entirely, making non-overrideable properties the
      default.
    * Make the `readOnly` modifier a no-op, and show a deprecation warning when it
      is used.

The warnings should explain the deprecation, and recommend that users do not
rely on setter behavior or opting-in to read only behavior.

# How We Teach This

In general, we can teach that computed properties are essentially cached native
getters/setters (with a few more bells and whistles). Once we have official
decorators in the framework, we can make this connection even more solid.

We should add notes on overridability, and we should scrub the guides of any
examples that make use of overriding directly and indirectly via `.readOnly()`.

# Drawbacks

Overriding is not a completely uncommonly used feature, and developers who have
become used to it may feel like it makes their code more complicated, especially
without any easy way to opt back in.

# Alternatives

We could convert `.readOnly()` into `.overridable()`, forcing users to opt-in
to overriding. Given the long timeline of this deprecation, it would likely be
better to work on making getters/setters transparent to decoration, and provide
a `@overridable` decorator either in Ember or as an independent package.
