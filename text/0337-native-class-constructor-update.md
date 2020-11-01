---
Start Date: 2018-06-14
RFC PR: https://github.com/emberjs/rfcs/pull/337
Ember Issue: https://github.com/emberjs/ember.js/pull/16795

---

# Native Class Constructor Update

## Summary

Update the behavior of EmberObject's constructor to defer object
initialization.

## Motivation

Using native class syntax with EmberObject has almost reached full feature
parity, meaning soon we'll be able to ship native classes and begin recommending
them. This will do wonders for the Ember learning story, and will bring us in
line with the wider Javascript community.

However, early adopters of native classes have experienced some serious
ergonomic issues due to the current behavior of the class constructor. The issue
is caused by the fact that properties passed to `EmberObject.create` are
assigned to the instance in the root class `constructor`. Due to the way that
native class fields work, this means that they are assigned _before_ any
subclasses' fields are assigned, causing subclass fields to overwrite any value
passed to `create`:

```js
class Foo extends EmberObject {
  bar = 'baz';
}

let foo = Foo.create({ bar: 'something different' });

console.log(foo.bar); // 'baz'
```

This has made adoption very difficult, and is a consistent stumbling block for
new users of native class syntax in Ember. Worse yet, it makes writing a codemod
for converting to native class syntax very difficult because we don't have a
clear target.

For instance, given the above class, how would we convert the class field? Let's
go through the various options:

```js
class Foo extends EmberObject {
  // Does not work, for the reasons described above
  bar = 'baz';

  // Does not cover all cases. If we did `Foo.create({ bar: false })` it would
  // still assign the default.
  bar = this.bar || 'baz';

  // This works, but is very verbose and not ideal
  bar = this.hasOwnProperty('bar') ? this.bar : 'baz';

  // This is one of the community accepted solutions, but it requires lodash
  bar = _.defaultTo(this.bar, 'baz');

  // This is another community accepted solution, but it requires
  // @ember-decorators/argument, which is a separate library
  @argument foo = 'bar';
}
```

None of these is ideal. Instead, we can change the behavior of the constructor
and the `create` method to circumvent this issue.

This change _would_ be a breaking change to the behavior of native classes
today, and a change from the previous class RFC. This will impact early adopters
and should be made with that in mind. It would _not_ be a change that breaks the
behavior of the community solutions to class fields mentioned above, and all
other changes would be relatively easy to create a safe codemod for (essentially
converting `constructor` -> `init` in affected classes), so the impact _should_
be minimal.

Because native classes never officially shipped as part of Ember's public API
(an announcement was not made, docs have not been written, etc), this RFC
proposes that the change would _not_ be considered a breaking change _for the
purposes of semver_. This would allow us to ship the change during the Ember v3
release cycle, and prevent more code from being built on top of the previous
behavior.

## Detailed design

One very important design constraint to making this change is that we _cannot_
break the behavior of EmberObject when used _without_ native classes. To do
this, we will leverage the fact that the static `create` method is the only
public way to create an instance of EmberObject.

Currently, the behavior of EmberObject is the following (simplified):

```js
class EmberObject {
  constructor(props) {
    // ..class setup things

    Object.assign(this, props);
    this.init();
  }

  static create(props) {
    let instance = new this(props);

    return instance;
  }
}
```

We can change it to the following (simplified):

```js
class EmberObject {
  constructor(props) {
    // ..class setup things
  }

  static create(props) {
    let instance = new this(props);

    Object.assign(instance, props);
    instance.init();

    return instance;
  }
}
```

This would assign the properties _after_ all of the class fields for any
subclasses have been assigned. Revisiting our previous example, the following
two class declarations would effectively be equivalent:

```js
const Foo = EmberObject.extend({
  bar: 'baz'
});

class Foo extends EmberObject {
  bar = 'baz';
}
```

Much easier to codemod! There are other subtle differences between native class
fields and EmberObject properties, such as the fact that class fields are
assigned each time a class is initialized, but these are easier to work around.

### Injections and the `init` hook

One side effect of this change is that injections will not be available on the
class instance during the `constructor` phase. This behavior is not very
commonly used - based on an informal community survey we found only a few usages
\- but it _does_ exist and have its use cases.

Figuring out the ideal behavior of injections during the constructor phase is
outside of the scope of this RFC, and is something that should be discussed in
future RFCs. For the time being, users can still rely on the `init` hook, which
will continue to be called after all injections and properties have been
assigned to the instance.

### `new EmberObject()`

It was previously possible to use `new` syntax with EmberObject. While this
was not considered public API, it has technically worked and been under test
since the early days of Ember, and may fall under the category of intimate API.
Ideally, we would deprecate this usage as a private/intimate API, which would
mean supporting it through the next LTS version, and dropping support after
(currently, this would mean dropping it at `v3.5.0`).

We can continue to support this behavior in a backwards compatible way while
deprecating it with one final tweak to the change above:

```js
const DEFER_INIT = new Symbol();

function initialize(instance, props) {
  Object.assign(instance, props);
  instance.init();
}

class EmberObject {
  constructor(props, maybeDefer) {
    // ..class setup things

    if (maybeDefer === DEFER_INIT) {
       return this;
    }

    deprecate('using `new` with EmberObject has been deprecated. Please use `create` instead.', false, {
      id: 'object.new-constructor',
      until: '3.5.0'
    });

    initialize(this, props);
  }

  static create(props) {
    let instance = new this(props, DEFER_INIT);
    initialize(instance, props);

    return instance;
  }
}
```

## How we teach this

If this PR is accepted, most of the major issues with classes will have been
resolved. We can begin working on a codemod to make converting easier, and move
toward officially making native classes a finalized part of the public API of
Ember. Pending decorators and class fields moving to a late enough stage in the
TC39 process, we can also begin converting the guides to use native class
syntax.

We can document the exact behavior of the new constructor in the API docs for
EmberObject. Most details won't have to change since this change only affects
native class syntax, which has not been documented much officially. We can also
demonstrate the behaviors of classes throughout the guides and API docs.

One thing we should make clear is that EmberObject will likely be deprecated
in the near future, and that ideally for non-Ember classes (things that aren't
Components, Services, etc.) users should drop EmberObject altogether and use
native classes only.

## Drawbacks

This would be a breaking change that could negatively affect early adopters.

## Alternatives

* We could leave the behavior as is, and choose a method for defaulting to
standardize on.

* We could make this change behind a feature flag and require users to opt-in
to the new behavior, like optional features that currently exist. This would
have to be a build time feature flag, since the area is very performance
sensitive. Given native classes are not yet public API, if we were to do this we
should probably still default to enabling the new behavior and recommending it
as the preferred path.

* We could not deprecate `new EmberObject` altogether, and instead only
deprecate passing properties to the constructor. While this would work as a
temporary solution, it may also encourage users to continue using EmberObject
instead of switching to native classes, which is ultimately the long term goal.

## Unresolved questions

How do we handle DI during the construction phase?
