---
Start Date: 2017-07-28
RFC PR: #240
Ember Issue: (leave this empty)

---

# Summary

This RFC aims to solidify the usage of ES2015 Classes as a public API of Ember so
that users can begin building on them, and projects like `ember-decorators` can
continue to push forward with experimental Javascript features. This includes:

* Making the class `constructor` function a public API
* Modifying some of the internals of `Ember.Object` to support existing features
and make the usage of ES Classes cross-compatible with `Ember.Object`

It does _not_ propose additions in the form of helpers or decorators, which should
continue to be iterated on in the community as the spec itself is finalized. It also
does not propose deprecating or removing existing functionality in `Ember.Object`.

# Motivation

The Ember Object model has served its purpose well over the years, but now that
ES Classes are becoming prevalent throughout the wider Javascript community
it is beginning to show its age. With class properties at stage 3 and decorators at
stage 2 in the TC39 process, classes are finally at a point where we can start
integrating them into Ember.

The [ember-decorators](https://github.com/rwjblue/ember-decorators) project has been
experimenting with using ES Classes and filling out the Ember feature-set,
allowing us to write Ember classes like so:

```javascript
export default class MyComponent extends Ember.Component {
  didInsertElement() {
    // do stuff
  }

  @computed
  get foo() {
    // do stuff
  }

  @action
  bar() {
    // do stuff
  }
}
```

Using classes makes Ember easier to teach and understand by normalizing it with
standard Javascript coding practices, and allows us to share code and solutions
with other frameworks and libraries. It also brings with it all the benefits of
ES Class syntax:

* More aligned with the greater Javascript community
* Ability to share code more easily with other libraries and frameworks
* Easier to statically analyze
* Cleaner and easier to read (subjective)

The Ember Object model already works extremely well with ES classes, as
demonstrated above, but there several failure scenarios. Furthermore, because
they are not officially supported as a public API, there is no guarantee
that they will continue to work well. Thus, this RFC seeks to solidify the
behavior of ES Classes so that the community can continue to experiment with
new Javascript features and build on a stable API.

# Detailed Design

Many of the standard features of Ember classes work out of the box today, either with
vanilla ES Classes or through `ember-decorators`, including:

* Inheritance
* Lifecycle hooks
* Computeds
* Injections
* Actions

However, the following features either do not exist or do not work as a
user familiar with `Ember.Object` would expect:

* Extending from ES Classes using `extend`
* Class properties
* Mixins
* Observers and events
* Merged and concatenated properties

These features will require changes to `Ember.Object`

## Extend

Currently, once a class is defined using ES Classes it is not possible for users
to extend it using the previous CoreObject style of writing and extending classes.
This can limit the rate of adoption because ES Classes would become a trapdoor -
once you begin using them, you must continue to use them. It would be a particularly
thorny issue for addon developers, who may design components which their users
expect to be able to extend and modify.

This RFC proposes that `extend` be fixed on ES Classes to make them fully
cross-compatible with the existing syntax. There are two general approaches to
making this work:

1. Modify CoreObject to use prototypes/ES Classes internally. This would bring
CoreObject more inline with ES Classes, but would be a significant internal change.

2. Modify CoreObject to have different behavior if it is extending an ES Class
using `extend`.

Both approaches should be explored and benchmarked to determine if there are an
significant advantages to one over the other.

## Class Properties

When using `Ember.Object.extend`, properties that are passed in on the object
are assigned to the prototype of the class:

```javascript
const Foo = Ember.Object.extend({ bar: 'baz' });
const foo = Foo.create();

console.log(Foo.prototype.bar) // 'baz'
foo.hasOwnProperty('bar') // false
```

This differs from the behavior of ES Class properties, which initialize their
value on the instance of the class.

```javascript
class Foo {
  bar = 'baz'
}

const foo = new Foo();

console.log(Foo.prototype.bar) // undefined
foo.hasOwnProperty('bar') // true
```

The above is essentially currently compiled down by Babel to the following:

```javascript
class Foo {
  constructor() {
    this.bar = 'baz';
  }
}
```

Property assignments like this are always done at the end of the constructor,
and given the requirement that `super` must always be called before properties
are assigned it is unlikely that this will change as the spec progresses.

While one might intuitively expect class properties to function the same in
ES Classes as they do with Ember Objects, this difference in behavior means that
class properties will always be assigned after properties passed into `create`
are initialized on the object, and thus will always win:

```javascript
const Foo = Ember.Object.extend({ testProp: 'default value' });

class Bar extends Ember.Object {
  testProp = 'default value'
}

const foo = Foo.create({ testProp: 'new value' });
const bar = Bar.create({ testProp: 'new value' });

console.log(foo.get('testProp')); // 'new value'
console.log(bar.get('testProp')); // 'default value'
```

This behavior makes sense when you consider that it is equivalent to assigning
values in `init` rather than on the object when it is defined. Rather than modify
`Ember.Object` to treat class properties as default values, this RFC proposes that
we accept the difference in behavior and utilize the constructor to allow users
to set default values, as in the following example:

```javascript
class Foo extends Ember.Object {
  constructor(props) {
    props.testProp = props.testProp || 'default value';

    super(props);
  }
}
```

This enforces a public API rather than allowing `create` to override values as
it pleases, and is more inline with the behavior of components in Glimmer today -
args that are passed into the class are distinguished from properties that are
defined on the class.

## Mixins

Mixins are a contentious part of both the Ember Object model and the wider
Javascript community - some swear by the pattern, and others believe it fundamentally
flawed. While Ember mixins are at the core of Ember Object, the fact is that
no standard solution for them has arisen in the wider Javascript community as
of yet.

Additionally, while concepts like computed properties, actions, and
service injection are either unique to Ember or highly dependent on implementation,
mixins can be implemented in a generic way which could be used across all of
Javascript, independent of one's framework or library of choice. With that in
mind, this RFC considers mixins out of scope and suggests that in the future Ember
users can choose to use a mixin library if it suits their needs.

It should also be noted that existing classes which have used mixins can still be
extended using ES Class syntax:

```javascript
const Mix = Ember.Mixin.create({ bar: 'baz' });
const Foo = Ember.Object.extend(Mix, { /* ... */ });

class Bar extends Foo { /* ... */ }

const bar = Bar.create();

console.log(bar.get('bar')); // 'baz'
```

## Observers and Events

Observers and events both fail to work properly when using ES Class syntax. The root
of the issue here is how `Ember.Object` works at a fundamental level, and will require
some refactoring to fix.

Currently, each time `Ember.Object.extend` is used, it stores the list of mixins and
objects passed in on a list which also contains the superclass's properties and mixins,
and so on. A class is then returned which has access to a closure variable, `wasApplied`:

```javascript
makeCtor = function() {
  wasApplied = false;

  return class {
    constructor() {
      if (!wasApplied) {
        this.proto();
      }
    }
  }
}
```

The `proto` function walks the chain of stored mixins, collapsing them into a single object
prototype the first time the class is created. It is during this walk that observers and
events listeners are applied and finalized, as well as merged and concatenated properties
applied (this will be touched on more in the next section).

Unfortunately, due to the nature of how observers and event listeners work, they cannot be
applied at class definition time without a class decorator. For example:

```javascript
const Foo = Ember.Object.extend({
  fooObserver: Ember.observer('foo', function() { /* ... */m })
});

class Bar extends Foo {
  fooObserver() { /* ... */ }
}
```

When `proto` walks the mixin chain for Foo, it will add an observer that triggers the
`fooObserver` function whenever `foo` changes. Bar, however, overloads the `fooObserver`
function with a function that is _not_ observed, and thus should not trigger (this is
analagous to how Ember Object's work today). Currently there is no time at which
Bar can inspect undecorated properties to determine if the superclass has already defined
them and if they are observed and thus should have the observer removed.

To fix this, the `wasApplied` state should be moved to the ember meta object on the
class itself, so that both Ember Objects and ES Classes can track if they have had it
applied. Additional logic will also need to be added to allow the current "squashing"
behavior of `proto` to work with Prototypes instead of a list of mixins as well.

## Merged and Concatenated Properties

Ember Objects currently have the ability to define special properties which are
merged or concatenated with their superclass when extended. This is most commonly
seen with `actions` and `classNames` among others.

As mentioned in the last section, merged and concatenated properties are also
combined during the `proto` "squash" phase, and so it is also broken in ES Classes
currently. This RFC proposes that their behavior also be fixed as part of the refactors
to Ember.Object.

# How We Teach This

The sole purpose of this RFC is to make the behavior of ES Classes within Ember a
public API so that projects like `ember-decorators` can continue to build and experiment
with confidence that the underlying behavior will not change. The Ember Object model
will remain exactly the same as today, and will continue to be the recommended path
for Ember users. Thus, we will not need to add new documentation for the time being.

# Drawbacks

* Making `constructor` a public API means we are solidifying the lifecycle of
objects, locking us into a particular sequence of events (`init` occurs within
the `super()` portion of the constructor).
* Lack of mixin support may make it difficult for mixin heavy codebases to utilize
ES Classes.
* ES Class features/usage such as getters and setters may confuse users in general
(getter functions will _appear_ to work, but without a `computed` decorator will
not update, etc.)

# Alternatives

* Class property initialization can be changed such that properties are initialized
after the constructor runs entirely, allowing them to be overwritten by values
passed to `create`

# Topics for Future RFCs

While working on this RFC, some issues were brought into focus regarding existing
features in CoreObject that are seen as problematic or unintuitive. In order to
avoid bikeshedding these have been slated for discussion in future RFCs, but the
discussion points have been included below.

## Merged and Concatenated Properties

Merged and concatenated properties are pain points for new Ember developers,
specifically because they give no lexical hint that they are special in any way.
Developers must know that these particular properties will be merged with the
superclass, and there is no way to opt out of this behavior.

With decorators, this same behavior can be accomplished in a much clearer and
more straightforward way:

```javascript
class FooComponent extends Ember.Component {
  @concatenated classNameBindings = ['foo']

  @computed
  get foo() { /* ... */ }

  @merged actions = {
    bar() { /* ... */ }
  }
}
```

They could also be accomplished more ergonomically with specialized decorators:

```javascript
class FooComponent extends Ember.Component {
  @className
  @computed
  get foo() { /* ... */ }

  @action
  bar() { /* ... */ }
}
```

This approach has two distinct advantages over the existing behavior:

1. It is less magical. The decorators indicate to new users that the properties are
special in some way, and ultimately they are just plain decorators, which are
compatible with ES Classes as a whole and can be reused anywhere.
2. It provides a way to opt out of the behavior. Currently, there is no easy
way to prevent properties which were marked to be merged from being merged,
meaning subclasses are stuck with the values that their superclass provided.

## Observers and Listeners

Observers and event listeners are a powerful pattern that saw a lot of usage in Ember 1.
However, it is now widely accepted that they are problematic when overused, and using
computed properties and lifecycle hooks are better patterns in most cases.

As such, rather than having events and observers turned on by default it may make
more sense to have them be opt-in APIs. This could be accomplished by making new
class decorators like so:

```javascript
@evented
class Foo extends Ember.Object {
  @on('init')
  onInit() {
    // do something
  }
}
```

Or it could be accomplished with new base classes that include the functionality:

```javascript
class Foo extends EventedObject {
  @on('init')
  onInit() {
    // do something
  }
}
```

# Unresolved questions

None currently
