- Start Date: 2019-02-19
- Relevant Team(s): Ember.js, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/451
- Tracking:

# Classic Class Owner Tunnel 🕳

## Summary

Make `getOwner` and explicit injections work in classic class constructors.

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

The [Native Class Constructor Update][1] RFC changed the way that classic
classes were constructed, and one major side effect of this was that it made
injections and the container inaccessible during the `constructor` of classes
that extend from `EmberObject`. This was a necessary change for other reasons,
but the lack of injections means that we still have to use `init` for various
use cases in several classes, including:

[1]: https://github.com/emberjs/rfcs/blob/master/text/0337-native-class-constructor-update.md

- `Service`
- `Route`
- `Controller`

These classes would otherwise be _completely free_ of any of the trappings of
the classic object model. As the Octane striketeam has begun updating the guides
for these, the fact that users must learn to use `init` with them has become a
pain point in the new guides. This is _especially_ painful given that
`GlimmerComponent` does _not_ have an `init` hook, meaning we can't teach users
to just use `init` everywhere either. Instead, users must learn that they have
to use one or the other depending on what type of class they're using, even in
completely new Ember Octane apps.

By tunneling the owner through to the root constructor, we can unify our
documentation and best practices around one recommendation. New Ember Octane
applications will be able to completely forgo `init`, and never learn about it
in the first place. Existing Ember apps will still have cases where it is
necessary (classic components, utility classes), but this is no worse a status
quo than where we are currently - some classes require `constructor`, and some
require `init`.

Finally, if we do this change, we could lint users to a place where they are not
using _any_ EmberObject APIs at all for any of these constructs. This means that
in the future, we _could_ deprecate them extending from `EmberObject` at all,
without forcing users onto new import paths. This deprecation is outside of the
scope of this RFC, but it sets us up to move forward with decoupling Ember from
the classic object model in the future.

## Detailed design

The "tunnel" itself is fairly simple. As described in the Constructor Update
RFC, this is how the `create` method on classes works currently:

```js
class EmberObject {
  constructor() {
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
class EmberObject {
  constructor(owner) {
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
affected. Some users may feel like there is some churn, especially TypeScript
users (sorry folks!)

These changes will also be backported to at least:

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

Supplemental guides should also be written to cover the nuances of using native
class syntax with either of these, including when and where to use `init`.
Additionally, lint rules should be made if possible that enforce that the user
is using the correct method. These lint rules may need some hinting in the form
of a decorator or other "trigger" that let's them know what type of class a
given class is:

```js
// This tell the linter that this is a classic component that needs to be upgraded
@todo('octane-upgrade')
export default class FooComponent extends ClassicComponent {
  // this throws an error because of the annotation, but will be fine once the
  // annotation is removed.
  constructor() {}
}
```

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
  around.
