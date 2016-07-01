- Start Date: 2016-06-11
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

With the goal of making significant performance improvements and of adding
public API to support use cases long-served by a private API, a new API of
`factoryFor` will be added to `ApplicationInstance` instances.

# Motivation

Ember's dependency injection container has long supported fetching a factory
that will be created with any injections present. Using the private API that
provided this support would allow an instance of the factory to be created
with initial values passed via `create`. For example:

```js
import Ember from 'ember';
const { Component, getOwner } = Ember;

export default Component.extend(
  init() {
    this._super(...arguments);
    let Factory = getOwner(this)._lookupFactory('logger:main');
    this.logger = Factory.create({ level: 'low' });
  }
});
```

In this API, the `Factory` is actually a subclass the main logger class. In
`_lookupFactory`, there are minimum two class `extend` calls:

* `MyClass = Ember.Object.extend(`
* `MyFactoryWithInjections = MyClass.extend(`
* `MyFactoryWithInjections.create(`

The middle extend only serves to implement dependency injection- merging the
properties passed to `create` with the injected ones. The double extend
of classes in Ember takes a toll on performance booting an app. This
poor design has kept the factory lookup API private.

Additionally the middle extend is a cache, and cleared between tests or
application instances. For example:

```
               +-------------------------------+
               |                               |
               |      /app/models/foo.js       |
               |                               |
               +-------------------------------+
                               |
              first test run   |    nth test run
              +----------------+---------------+
              |                                |
              v                                v
   +---------------------+          +--------------------+
   |resolve('model:foo') |   ===    |resolve('model:foo')|
   +---------------------+          +--------------------+
              |                                |
              |                                |
              v                                v

     extend(injections)               extend(injections)

              |                                |
              |                                |
              |                                |
              v                                v
+--------------------------+     +---------------------------+
|lookupFactory('model:foo')| !== |lookupFactory('model:foo') |
+--------------------------+     +---------------------------+
```

However despite these issues being able to instantiate a factory with initial properties is useful for
a number of scenarios and is a common practice
in a number of important addons.

* [ember-cart](https://github.com/DockYard/ember-cart) uses the functionality to create model objects without
  tying them to the store [example a](https://github.com/DockYard/ember-cart/blob/c01eb22eaf2e97f8c80481c3174d4be917e476a9/tests/dummy/app/controllers/application.js#L16),
  [example b](https://github.com/DockYard/ember-cart/blob/c01eb22eaf2e97f8c80481c3174d4be917e476a9/tests/dummy/app/models/dog.js#L11)
* Ember-Data's [`modelFactoryFor`](https://github.com/emberjs/data/blob/54ea432b1cbb0d1231d9a0454b09d3b3a0bc2533/addon/-private/system/store.js#L1868)

Many use-cases can be handled with a convention of calling a `setup` method
on object instances after their creation by the container. However this convention
would be at odds with common OO patterns (where you use the constructor to
setup an instance) and performance best practices (which prefer to see
properties defined on an object during its construction).

# Detailed design

This feature will be added in three steps.

1. Introduce `ApplicationInstance#factoryFor` in Ember 2.8
2. Deprecate `ApplicationInstance#_lookupFactory` in Ember 2.8, remove in 2.9
3. Release a polyfill addon for this feature

#### Introduce `ApplicationInstance#factoryFor` in Ember 2.8

A new API will be introduced. This API will return both the original base
class resisted (or resolved) by the container, and will also return a function
to generate a dependency-injected instance. For example:

```js
import Ember from 'ember';
const { Component, getOwner } = Ember;

export default Component.extend(
  init() {
    this._super(...arguments);
    let factory = getOwner(this).factoryFor('logger:main');
    this.logger = factory.create({ level: 'low' });
  }
});
```

Unlike `_lookupFactory`, `factoryFor` will not return an extended class with
DI applied. Instead it will return a factory object with two properties:

```js
// factoryFor returns:
let {

  // a function taking an argument of initial properties passed to the object
  // and returning an instance
  create,

  // The class registered into (or resolved by) the container
  class

} = owner.factoryFor('type:name');
```

This API should meet two requirements of the use-cases described in
"Motivation":

* The implementation of `create` can diverge away from double extend. The
  class of an object instantiated via `_lookupFactory(name).create()`
  and `factoryFor(name).create()` may not be the same, even given the
  same factory being registered into the container.
* The presence of `class` will make it easy to identify the base class of the
  factory at runtime.

For example today's `_lookupFactory` creates an inheritance structure like
the following:

```
                    Current:
       +-------------------------------+
       |                               |
       |      /app/models/foo.js       |
       |                               |
       +-------------------------------+
                       |
                       |
                       |
                       v
            +--------------------+
            |  Class[model/Foo]  |
            +--------------------+
                       |
                       |
                       |
       first test run  |   nth test run
           +-----------+----------+
           |                      |
           |                      |
           |                      |
           v                      v
+--------------------+ +--------------------+
|     subclass of    | |     subclass of    |
|  Class[model/Foo]  | |  Class[model/Foo]  |
+--------------------+ +--------------------+
```

This means, between test runs 2 instances of `model:foo` will have a common
shared ancestor the grandparent `Class[model/Foo]`.

We propose is to remove the intermediate subclass and instead have a generic
factory object, which holds the injections and allows for injected instances
to be created. The resulting object graph would look something like this:
(between test runs 2 instance of `model:foo` will have a shared ancestor
(parent) `Class[model/Foo]`.

```
                  Proposed:
      +-------------------------------+
      |                               |
      |      /app/models/foo.js       |
      |                               |
      +-------------------------------+
                      |
                      |
                      |
                      v
           +--------------------+
           |  Class[model/Foo]  |
           +--------------------+
                      |
                      |
                      |
      first test run  |   nth test run
           +----------+-----------+
           |                      |
           |                      |
           |                      |
           v                      v
+--------------------+ +--------------------+
|     Factory of     | |     Factory of     |
|  Class[model/Foo]  | |  Class[model/Foo]  |
+--------------------+ +--------------------+
```

This results in `instance` direct `parent` being shared between test runs. More
specially this means any instance state stored on the `parent` will leak
between tests.

#### Deprecate `ApplicationInstance#_lookupFactory` in Ember 2.8, remove in 2.9

In 2.8 (an LTS release) `_lookupFactory` will be deprecated with a message
suggesting a migration to the new API. In 2.9 the API will be removed.

#### Release a polyfill addon for this feature

A polyfill addon, similar to [ember-getowner-polyfill](https://github.com/rwjblue/ember-getowner-polyfill)
will be released for this feature. This polyfill will provide the `factoryFor`
API going back to at least 2.4, provide the API and silence the deprecation
in 2.8, and be a no-op in 2.9+ where Ember provides the `factoryFor` API.

# How We Teach This

This feature should be introduced along side `lookup` in the
[relevant guide](https://guides.emberjs.com/v2.6.0/applications/dependency-injection/).
The return value of `factoryFor` should be taught as a POJO and not as
an extended class.

# Drawbacks

The main drawback to this solution is the removal of double extend. Double
extend is a performance troll, however it also means if a single class is registered
multiple times each `_lookupFactory` returns a unique factory. It is plausible
that some use-case relying on this behavior would get trolled in the migration
to `factoryFor`, however it is unlikely.

For example these cases where state is stored on the factory would no
longer be viable:

- ember-model
  - https://github.com/ebryn/ember-model/blob/master/packages/ember-model/lib/model.js#L404
  - https://github.com/ebryn/ember-model/blob/master/packages/ember-model/lib/model.js#L457
  - https://github.com/ebryn/ember-model/blob/master/packages/ember-model/lib/model.js#L723-L725
-  ember-data
  - As far as I can tell, the big issues have been resolved!
  - if attrs change between test runs (seems very unlikely) then https://github.com/emberjs/data/blob/387630db5e7daec6aac7ef8c6172358a3bd6394c/addon/-private/system/model/attr.js#L57 would be affected
- people using:
  - `lookupFactory(x).reopen` / `reopenClass` at runtime (or test time to monkey patch code)
  - `lookupFactory(x).something = value`

# Alternatives

No other designs have been seriously considered.

We have considered the possibility that removing `_lookupFactory` in 2.9
(something LTS technically permits) would be too aggressive for the
community of addons. Providing a polyfill is part of the strategy to handle
this change.

However if the removal still proves too difficult, an alternative strategy
is possible:

* Add `factoryFor` in 2.8
* Deprecate `_lookupFactory` in 2.8, remove it in 3.0
* Introduce a polyfill add for `factoryFor` valid w/ 2.4-3.0
* In 2.9, migrate the implementation of `_lookupFactory` to be based on
  `factoryFor`. There may be some slight behavior change, or performance
  regression when using this API.

# Unresolved questions

Are there any use-cases for the double extend not considered?
