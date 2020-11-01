---
Start Date: 2016-06-11
RFC PR: #150
Ember Issue: #14360

---

# Summary

With the goal of making significant performance improvements and of adding
public API to support use cases long-served by a private API, a new API of
`factoryFor` will be added to `ApplicationInstance` instances.

# Motivation

Ember's dependency injection container has long supported fetching a factory
that will be created with any injections present. Using the private API that
provided this support allows an instance of the factory to be created
with initial values passed via `create`. For example:

```js
// app/logger/main.js
import Ember from 'ember';

export default Ember.Logger.extend({
  someService: Ember.inject.service()
});
```

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

In this API, the `Factory` is actually a subclass the original main logger
class. When `_lookupFactory` is called, an additional `extend` takes place
to add any injections (such as `someService` above). The class/object setup
looks like this:

* In the module: `MyClass = Ember.Object.extend(`
* In `_lookupFactory`: `MyFactoryWithInjections = MyClass.extend(`
* And when used: `MyFactoryWithInjections.create(`

The second call to `extend` implements Ember's owner/DI
framework and permits `someService` to be resolved later. The "owner" object
is merged into the new `MyFactoryWithInjections` class along with any
registered injections.

This "double extend" (once at define time, once at `_lookupFactory` time)
takes a toll on performance booting an app. This design flaw has motivated
a desire to keep `_lookupFactory` private.

The `MyFactoryWithInjections` class also features as a cache. Because it is
attached to the owner/container, it is cleared between test runs or
application instances. To illustrate, this flow-chart shows how
`MyFactoryWithInjections` diverges between tests:

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

Despite the design flaws in this API, it does fill a meaningful role in
Ember's DI solution. Use of the private API is common. Some examples:

* [ember-cart](https://github.com/DockYard/ember-cart) uses the functionality to create model objects without
  tying them to the store [example a](https://github.com/DockYard/ember-cart/blob/c01eb22eaf2e97f8c80481c3174d4be917e476a9/tests/dummy/app/controllers/application.js#L16),
  [example b](https://github.com/DockYard/ember-cart/blob/c01eb22eaf2e97f8c80481c3174d4be917e476a9/tests/dummy/app/models/dog.js#L11)
* Ember-Data's [`modelFactoryFor`](https://github.com/emberjs/data/blob/54ea432b1cbb0d1231d9a0454b09d3b3a0bc2533/addon/-private/system/store.js#L1868)

The goal of this RFC is to create a public API for fetching factories with
better performance characteristics than `_lookupFactory`.

# Detailed design

Throughout this document I reference Ember 2.12 as it is the next LTS at writing. This
proposal may ship for 2.12-LTS or be bumped to the next LTS.

This feature will be added in these steps.

1. In Ember introduce a `ApplicationInstance#factoryFor` based on
   `_lookupFactory`. It should be documented that certain behaviors
   inherent to "double extend" are not supported. In development builds
   and supporting browsers, wrap return values in a Proxy. The proxy should
   throw an error when any property besides `create` or `class` is accessed.
   `class` must return the registered factory, not the double extended factory.
2. In the same release add a deprecation message to usage of `_lookupFactory`.
   As this API is intimate it must be maintained through at least one LTS
   release (2.12 at this writing).
3. In 2.13 drop `_lookupFactory` and migrate the `factoryFor` implementation to avoid
   "double-extend" entirely.

Additionally, a polyfill will be released for this feature supporting prior
versions of Ember.

#### Design of `ApplicationInstance#factoryFor`

A new API will be introduced. This API will return both the original base
class registered into or resolved by the container, and will also return a function
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

* Because `factoryFor` only returns a `create` method and reference to the
  original class, its internal implementation can diverge away from the
  "double extend". A side-effect of this is that the
  class of an object instantiated via `_lookupFactory(name).create()`
  and `factoryFor(name).create()` may not be the same, given the
  same original factory.
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

Between test runs 2 instances of `model:foo` will have a common
shared ancestor the grandparent `Class[model/Foo]`.

This implementation of `factoryFor` proposes to remove the intermediate
subclass and instead have a generic
factory object which holds the injections and allows for injected instances
to be created. The resulting object graph would look something like this:

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

With `factoryFor` instances of `model:foo` will share a common constructor.
Any state stored on the constructor would of course leak between the tests.

An example implementation of `factoryFor` can be reviewed [on this GitHub
comment](https://github.com/emberjs/rfcs/issues/125#issuecomment-193827658).

##### Implications for `owner.register`

Currently, factories registered into Ember's DI system are required to
provide an `extend` method. Removing support for extend-based DI in `_lookupFactory`
will permit factories without `extend` to be registered. Instead factories
must only provide a `create` method. For example:

```js
let factory = {
  create(options={}) {
    /* Some implementation of `create` */
    return Object.create({});
  }
};
owner.register('my-type:a-factory', factory);
let factoryWithDI = owner.factoryFor('my-type:a-factory');

factoryWithDI.class === factory;
```

##### Development-mode Proxy

Because many developers will simply re-write `_lookupFactory` to `factoryFor`,
it is important to provide some aid and ensure they actually complete the
migration completely (they they avoid setting state on the factory). A proxy
wrapping the return value of `factoryFor` and raising assertions when any
property besides `create` or `class` is accessed will be added in development.

Additionally, using `instanceof` on the result of `factoryFor` should be
disallowed, causing an exception to be raised.

A good rule of thumb is that, in development, using anything besides `class` or
`create` on the return value of `factoryFor` should fail with a helpful message.

##### Releasing a polyfill

A polyfill addon, similar to [ember-getowner-polyfill](https://github.com/rwjblue/ember-getowner-polyfill)
will be released for this feature. This polyfill will provide the `factoryFor`
API going back to at least 2.8, provide the API and silence the deprecation
in versions before `factoryFor` is available, and be a no-op in versions where
`factoryFor` is available.

# How We Teach This

This feature should be introduced along side `lookup` in the
[relevant guide](https://guides.emberjs.com/v2.6.0/applications/dependency-injection/).
The return value of `factoryFor` should be taught as a POJO and not as
an extended class.

#### Example deprecation guide: Migrating from `_lookupFactory` to `factoryFor`

Ember owner objects have long provided an intimate API used to
fetch a factory with dependency injections. This API, `_lookupFactory`, is deprecated
in Ember 2.12 and will be removed in Ember 2.13. To ease the transition to this
new public API, a polyfill is provided with support back to at least Ember 2.8.

`_lookupFactory` returned the class of resolved factory extended with
a mixin containing its injections. For example:

```js
let factory = Ember.Object.extend();
owner.register('my-type:a-name', factory);
let klass = owner._lookupFactory('my-type:a-name');
klass.constructor.superclass === factory; // true
let instance = klass.create();
```

`factoryFor` instead returns an object with two properties: `create` and `class`.
For example:

```js
let factory = Ember.Object.extend();
owner.register('my-type:a-name', factory);
let klass = owner.factoryFor('my-type:a-name');
klass.class === factory; // true
let instance = klass.create();
```

A common use-case for `_lookupFactory` was to fetch an factory with
specific needs in mind:

* The factory needs to be created with initial values (which cannot be
  provided at create-time via `lookup`.
* The instances of that factory need access to Ember's DI framework (injections,
  registered dependencies).

For example:

```js
// app/widgets/slow.js
import Ember from 'ember';

export default Ember.Object.extend({
  // this instance requires access to Ember's DI framework
  store: Ember.inject.service(),

  convertToModel() {
    this.get('store').createRecord('widget', {
      widgetType: 'slow',
      name, canWobble
    });
  }

});
```

```js
// app/services/widget-manager.js
import Ember from 'ember';

export default Ember.Service.extend({

  init() {
    this.set('widgets', []);
  },

  /*
   * Create a widget of a type, and add it to the widgets array.
   */
  addWidget(type, name, canWobble) {
    let owner = Ember.getOwner(this);
    // Use `_lookupFactory` so the `store` is accessible on instances.
    let WidgetFactory = owner._lookupFactory(`widget:${type}`);
    let widget = WidgetFactory.create({name, canWobble});
    this.get('widgets').pushObject(widget);
    return widget;
  }

});
```

For these common cases where only `create` is called on the factory, migration
to `factoryFor` is mechanical. Change `_lookupFactory` to `factoryFor` in the
above examples, and the migration would be complete.

##### Migration of static method calls

Factories may have had static methods or properties that were being accessed
after resolving a factory with `_lookupFactory`. For example:

```js
// app/widgets/slow.js
import Ember from 'ember';

const SlowWidget = Ember.Object.extend();
SlowWidget.reopenClass({
  SPEEDS: [
    'slow',
    'verySlow'
  ],
  hasSpeed(speed) {
    return this.SPEEDS.contains(speed);
  }
});

export default SlowWidget;
```

```js
let factory = owner._lookupFactory('widget:slow');
factory.SPEEDS.length; // 2
factory.hasSpeed('slow'); // true
```

With `factoryFor`, access to these methods or properties should be done via
the `class` property:

```js
let factory = owner.factoryFor('widget:slow');
let klass = factory.class;
klass.SPEEDS.length; // 2
klass.hasSpeed('slow'); // true
```

# Drawbacks

The main drawback to this solution is the removal of double extend. Double
extend is a performance troll, however it also means if a single class is registered
multiple times each `_lookupFactory` returns a unique factory. It is plausible
that some use-case relying on this behavior would get trolled in the migration
to `factoryFor`, however it is unlikely.

For example these cases where state is stored on the factory would no
longer be scope to one instance of the owner (like one test). Instead, setting
a value on the class would set it on the registered class.

Some real-world examples of setting state on the factory class:

- ember-model
  - https://github.com/ebryn/ember-model/blob/master/packages/ember-model/lib/model.js#L404 and https://github.com/ebryn/ember-model/blob/master/packages/ember-model/lib/model.js#L457
    with `factoryFor` will increment a shared counter across application and
    container instances.
  - https://github.com/ebryn/ember-model/blob/master/packages/ember-model/lib/model.js#L723-L725
    would also set properties on the base `Ember.Model` factory instead of
    an extension of that class.
- ember-data
  - If attrs change between test runs (seems very unlikely) then https://github.com/emberjs/data/blob/387630db5e7daec6aac7ef8c6172358a3bd6394c/addon/-private/system/model/attr.js#L57
    would be affected. The CP of `attributes` will have a value cached on the
    factory, and where with `_lookupFactory`'s double-extend the cache would be
    on the extended class, in `factoryFor` that CP cache will be on the
    class registered as a factory.
- Any other of the following:
  - `lookupFactory(x).reopen` / `reopenClass` at runtime (or test time to monkey patch code)
  - `lookupFactory(x).something = value`

# Alternatives

More aggressive timelines have been considered for this change.

However we have considered the possibility that removing `_lookupFactory` in 2.13
(something LTS technically permits) would be too aggressive for the
community of addons. Providing a polyfill is part of the strategy to handle
this change.

# Unresolved questions

Are there any use-cases for the double extend not considered?
