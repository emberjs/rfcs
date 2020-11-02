---
Start Date: 2019-02-19
Relevant Team(s): Ember.js, Ember Data, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/451
Tracking: https://github.com/emberjs/rfc-tracking/issues/34

---

# Injection Parameter Normalization

## Summary

Normalize on passing the `owner` as the first parameter to the constructor for
the following built in framework classes:

- `GlimmerComponent`
- `EmberComponent`
- `Service`
- `Route`
- `Controller`
- `Helper`

Along with the following Ember Data classes:

- `Model`
- `Adapter`
- `Serializer`
- `Transform`

## Terminology

- _Explicit injections_ are injections which are defined in the class body using
  the `inject` APIs:

  ```js
  import Component from '@ember/component';
  import { inject as service } from '@ember/service';

  export default Component.extend({
    store: service(),
  });
  ```

  The are _explicit_ because they don't require any knowledge of the system to
  outside of the class itself to know they exist.

- _Implicit injections_ are injections which are defined using the container
  APIs directly, often in initializers:

  ```js
  import Application from '@ember/application';

  Application.initializer({
    name: 'inject-session',

    initialize() {
      // Inject the session service onto all factories of the type 'controller'
      // with the name 'session'
      App.inject('controller', 'session', 'service:session');
    },
  });
  ```

  They are implicit because they require knowledge of the context
  of the class to know whether or not they exist, simply looking at the class
  body (without looking at method logic) will not hint at their existence. The
  canonical example here is the Ember Data `store`, which is implicitly injected
  into all routes and controllers.

## Motivation

The introduction of native class syntax in Ember has recently exposed some of
the inner-workings and expectations of Ember's Dependency Injection (DI) system.
Specifically, it is now possible to write code that can run _before_
dependencies are injected in some base classes, such as Services and
Controllers. Currently, users must use the `init` hook in these classes if they
wish to run setup code that accesses injections, but this is somewhat confusing
since `init` has historically been taught as the same as the `constructor` in
native classes.

Glimmer components made the decision to break from this pattern, and instead
pass the DI Owner as the first parameter to the constructor. They then set it
using `setOwner` in the base class, making explicit injections available during the
constructor, and to class field initializers.

So far this has worked pretty well in practice:

- Glimmer components have just 2 lifecycle hooks, which makes them simpler to
  understand and learn about.
- We don't have to teach the differences between `constructor` and `init`, when
  to use one or the other, and debugging issues when the two have mixed usage
  throughout the class hierarchy
- We don't have to worry about explaining the timings/lifecycle of the container
  and the way it constructs classes in order to explain why these are separate.

This RFC seeks to normalize this contract for all _Ember base classes_ - that
is, framework classes that are provided by Ember:

- `GlimmerComponent`
- `EmberComponent`
- `Service`
- `Route`
- `Controller`
- `Helper`

Along with framework clases provided by Ember Data:

- `Model`
- `Adapter`
- `Serializer`
- `Transform`

This RFC does **_not_** aim to provide a single contract for _all_ classes
registered on the container, in perpetuity. This would lock us into a tight
coupling between the container and constructors for objects that are registered,
and wouldn't provide much flexibility in the future.

Instead, we believe we should continue exploring APIs for generalizing the way
DI is configured for a given base class. It could be done via custom managers,
or via decoration, like the [Injection Hook Normalization
RFC](https://github.com/emberjs/rfcs/pull/467). When these APIs are fully
rationalized and accepted, we'll update Ember's base classes to use them to
specify the owner injection like any other user class could.

## Detailed design

This RFC has 2 major parts:

1. The contract that we'll uphold for dependency injection in _Ember base
   classes_.
2. The implementation of that contract for existing base classes (the tunnel).

### Dependency Injection Contract

For all Ember base classes created by the container, such as `GlimmerComponent`,
`Service`, `Controller`, etc. we will:

1. Pass the owner as the first parameter when constructing the class.
2. Set the owner with `setOwner` in the base class constructor.

This will make explicit injections available during the `constructor` method of
the class, and for access by class field initializers.

This contract _only_ applies to Ember base classes and framework objects, and
classes that extend `EmberObject`. It does _not_ apply to arbitrary
classes that are created and registered in the container.

### Implementation

The "tunnel" itself is fairly simple. As described in the Constructor Update
RFC, this is how the `create` method on framework classes works currently:

```js
class Service extends EmberObject {
  constructor() {
    super();
    // ..class setup things
  }

  static create(args) {
    let instance = new this();

    Object.assign(instance, args);
    instance.init();

    return instance;
  }
}
```

We would update this to the following:

```js
class Service extends EmberObject {
  constructor(owner) {
    super();
    setOwner(this, owner);
    // ..class setup things
  }

  static create(args) {
    let owner = args ? getOwner(args) : undefined;
    let instance = new this(owner);

    Object.assign(instance, args);
    instance.init();

    return instance;
  }
}
```

Now, when any subclass's constructor code is run it will have the owner
available, which in turn makes all _explicit_ injections work (they use
`getOwner` under the hood).

However, _implicit_ injections will still only be available during `init`,
because they are passed in and assigned as `args`. This RFC proposes that rather
than attempting to fix implicit injections, we create development-mode
assertions for them which throw if a user attempts to use them during the
`constructor`, before they are assigned. This will give a helpful error message
that instructs them to add the injection explicitly (ideally), or to use `init`.

### Backwards Compatibility

This change is backwards compatible, so existing applications will not be
affected. These changes will also be backported to at least:

- lts-3.8
- lts-3.4

via the `ember-native-class-polyfill`, which currently supports polyfilling to
Ember@3.4. If possible, that range will be extended to the last v2 LTS versions.

## How we teach this

This change would take some burden off of the guides for _new_ Ember users,
post-Octane, since it would simplify them. New documentation should only refer
to `constructor` when talking about native class syntax, and should guide users
toward using `constructor` over `init`.

For existing apps and upgrade documentation, the distinction needs to be made
clear about the two types of classes that _should_ still use `init`:

- Classic Components
- Utility Classes (e.g. user defined classes that extend `EmberObject`)

These two require `init` if users need access to component args or create args,
respectively.

The main guides will recommend that users refactor these classes entirely rather
than convert them to native classes. Classic components should become Glimmer
components, and utility classes should be refactor _away_ from extending
`EmberObject`.

The [`@classic`](https://github.com/emberjs/rfcs/pull/468) decorator will also
provide a way to guide users toward the correct usage, based on whether they are
in "classic" mode or "octane" mode. We will be able to provide linting and
warnings/assertions to prevent users from accidentally using `init` when they
should have used `constructor`, and vice-versa.

## Drawbacks

- More churn in the ecosystem, early adopters of classes already switched from
  `constructor` -> `init`, switching back would be painful.
- Where to use `init` and where to use `constructor` may be a bit less clear
  after. This was already a concern with `GlimmerComponent`, but it may be more
  problematic if there are more exceptions.

## Alternatives

- Add an `init` hook to `GlimmerComponent` to unify it with the classic classes.
  This could be confusing to users of `GlimmerComponent` (why do injections
  work in GC but not any other class constructor? Why does GC have `init` _and_
  `constructor`?)
- Keep using `init` for classic classes for the indefinite future, and teach
  around it.
