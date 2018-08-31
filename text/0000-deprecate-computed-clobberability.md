- Start Date: 2018-08-30
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Deprecate computed clobberability and `computed().readOnly()` in favor of
read-only computeds as the default.

# Motivation

Computed properties have existed in Ember long before class syntax and native
accessors (getters and setters) were readily available, and as such they have a
few notable behavioral differences. As we move toward adopting native class
syntax and using a decorator-based form of computeds, it makes sense to
reconcile these differences so that users can expect them to work the same as
their native counterparts.

The main and most notable difference this RFC seeks to deprecate is computed
clobberability. There are some other notable differences, including the caching
behavior of the `return` value of setter functions, which may be addressed in
future RFCs.

## Clobberability

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

By constrast, computed properties without setters will be "clobbered" when they
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
prevents this clobberability.

# Transition Path

This RFC proposes that `readOnly` properties become the default, and that in
order to clobber users must opt in by defining their own setters:

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

Most computed macros are clobberable by default, the exception being `readOnly`.
This RFC proposes that all computed macros with the exception of `reads` would
become read only by default. The purpose of `reads` is to _be_ clobberable, so
its behavior would remain the same.

## Decorator Interop

It may be somewhat cumbersome to write clobbering functionality or add proxy
properties when clobbering is needed. In an ideal world, computed properties
would modify accessors transparently so that they could be composed with other
decorators, such as an `@clobberable` decorator:

```js
class Foo {
  @clobberable
  @computed('_value')
  get bar() {
    return this._value;
  }

  @clobberable
  @and('baz', 'qux')
  quux;
}
```

Currently this is not possible as computed properties store their getter/setter
functions elsewhere and replace them with a proxy getter and the mandatory
setter assertion, respectively. In the long term, making computeds more
transparent in this way would be ideal, but it is out of scope for this RFC.

# How We Teach This

In general, we can teach that computed properties are essentially cached native
getters/setters (with a few more bells and whistles). Once we have official
decorators in the framework, we can make this connection even more solid.

We should add notes on clobberability, and we should scrub the guides of any
examples that make use of clobbering directly and indirectly via `.readOnly()`.

# Drawbacks

Clobbering is not a completely uncommonly used feature, and developers who have
become used to it may feel like it makes their code more complicated, especially
without any easy way to opt back in.

# Alternatives

We could convert `.readOnly()` into `.clobberable()`, forcing users to opt-in
to clobbering. Given the long timeline of this deprecation, it would likely be
better to work on making getters/setters transparent to decoration, and provide
a `@clobberable` decorator either in Ember or as an independent package.
