---
Start Date: 2018-10-20
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/408
Tracking: https://github.com/emberjs/rfc-tracking/issues/7

---

# Decorators

## Summary

Native classes are now officially supported in Ember, but currently their usage
is very limited. Core Ember features such as computed properties, actions, and
service injections have no publicly supported APIs to be used with native class
syntax.

Decorators provide a way to abstract functionality and improve the developer
experience of working with native classes.  This RFC outlines the implementation
and rollout plan for bringing decorators to Ember's computed properties (and
other behavior) for use in native classes.

### A Note on Decorator Stability

[Decorators](https://github.com/tc39/proposal-decorators) are important to
adopting native class syntax. They are a formalization of the patterns we have
been using as a community for years, and it will not be possible to use native
classes ergonomically without them unless a number of major concepts (computed
properties, injections, actions) are rethought. That said, as of today (01/03/19)
decorators are still a [_stage 2_ proposal](https://tc39.github.io/process-document)
in TC39, which means that while they are fully defined as a spec, they are not
yet considered a candidate for inclusion in the language, and may have
incremental changes that could be breaking if/when moved to stage 3. As such,
merging support for them now would pose some risk. Additionally, [class
fields](https://github.com/tc39/proposal-class-fields) are also required for
effective use of decorators, and while they are stage 3 in the process, they
have not yet been fully accepted either.

Ember cannot guarantee that the spec won't change, and such changes cannot apply
to Ember's normal semver guarantees. But it can make the following guarantees:

1. If there are changes to the spec, and it ***is*** possible to avoid changing
   the public APIs of decorators, then Ember will make the changes necessary to
   avoid public API changes.

2. If there are changes to the spec, and it ***is not*** possible to avoid
   breaking changes, Ember will minimize the changes as much as possible, and
   will provide a codemod to convert from the previous version of the spec to
   the next.

3. If the spec is dropped from TC39 altogether, Ember would have to continue to
   provide support for decorators via babel transforms until they are deprecated
   following the standard RFC process, and removed according to SemVer. Reverse
   codemods which translate decorators and native class syntax back to classic
   class syntax _will_ be made, and alternatives for native class syntax will be
   explored.

4. Classic class syntax will continue to be supported _at least_ until these
   features have been stabilized in the JavaScript language, to allow us to
   revert these changes if necessary. They will most likely be supported for
   longer to allow a smooth transition for users who do not want to adopt native
   classes until they are completely stable.

This RFC is being made with the assumption that decorators will be moved to
stage 3 in the near future, _before_ this RFC is implemented in Ember,
dramatically reducing the risk of adopting decorators. If this RFC is accepted
and decorators are not advanced in a timely manner, a followup RFC should be
made to determine whether or not decorators should be adopted in stage 2, and
what the support for them would look like.

## Terminology

For the purposes of this RFC, we'll use the following terminology:

* The **Octane programming model** refers to the new programming model
  established by the Ember Octane edition. It includes _native classes_,
  _tracked properties_ and _Glimmer components_, and more generally refers to
  features that will be considered _core to Ember_ in the future.
* The **classic programming model** refers to the traditional programming model.
  It includes _classic classes_, _computed properties_, _event listeners_,
  _observers_, _property notifications_, and _classic components_, and more
  generally refers to features that will not be central to Ember Octane.
* **Native classes** are classes defined using the JavaScript `class` keyword
* **Classic classes** are classes defined by subclassing from `EmberObject`
  using the static `extend` method.

## Motivation

Native JavaScript class syntax has been evolving for the past three years, filling in
the cracks and providing better, more standardized ways to write classes for the
web. They will be a key part of the Octane programming model, and the ES Classes
RFC was the first step toward enabling Ember users to use native class syntax,
but there are still key features of Ember that are not usable with `class`
syntax today, including *computed properties*, *actions*, and *injections*.

These features cannot be used ergonomically with native classes as it stands.
The only options are to either use Ember's `defineProperty` function directly,
or to define these values in an anonymous class. Both of these options are hard
to read, and will be difficult to codemod in the future:

```js
// Using define property
import { computed, defineProperty } from '@ember/object';

class Person {
  constructor() {
    this.firstName = 'Melanie';
    this.lastName = 'Sumner';
  }
}

// define a computed property
defineProperty(
  Person.prototype,
  'fullName',
  computed('firstName', 'lastName', {
    get() {
      return `${this.firstName} ${this.lastName}`;
    }
  })
);

// Using intermediate extends
import EmberObject, { computed } from '@ember/object';

class Person extends EmberObject.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  })
}) {
  constructor() {
    super(...arguments);
    this.firstName = 'Melanie';
    this.lastName = 'Sumner';
  }
}
```

Another method which was used for some time was to assign these values using
class field initializers. This practice is problematic however as it creates a
new instance of the computed property per _instance_ of the class, and does not
work with native getters either:

```ts
import EmberObject, { computed } from '@ember/object';

export default class Person extends EmberObject {
  firstName = 'Melanie';
  lastName = 'Sumner';

  fullName = computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  });
}

new Profile().fullName; // returns the CP instance, not 'Melanie Sumner'
```

The missing piece of functionality that we need here are _decorators_, and in
fact that is not a coincidence. The roots of the current TC39 decorator proposal
can be traced back to Ember (Yehuda having worked on the first few drafts of it)
specifically because computed properties, observers, and so on _are_ decorators.
We've been using them for years in Ember, just with a non-standard,
slightly-less-clean syntax.

```ts
import Component from '@ember/component';
import { computed } from '@ember/object';

export default class Profile extends Component {
  firstName = 'Melanie';
  lastName = 'Sumner';

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

Native decorators bring a more natural way of declaring computed properties to
native classes than the current computed macros.

## Prior Art

The [Ember Decorators](http://ember-decorators.github.io/ember-decorators/)
project has been experimenting with using decorators within Ember for some time
now, with the goal of reaching feature parity with the classic object model, and
the learnings from that project will be used to inform the API design in this
RFC.

## Detailed design

This RFC proposes that:

1. `Ember.computed`, the `inject` macros, and the computed property macros be
   updated to return a standard JavaScript decorator. Internally, the classic
   object model will be made aware of these decorators, and know how to apply
   them. This will allow the same exact imports to continue working as both
   native decorators and classic decorators. This decorator _will_ be compatible
   with classes that extend from `EmberObject` and classes which do not.
2. A new `@action` decorator be added to allow the definition of actions on
   classes. This decorator will _only_ be compatible with classes that are
   action handlers.

This does leave out some features from the classic programming model which are
currently provided by Ember Decorators. This is both to minimize decorators' API
surface area, and because they will not be a major part of Ember Octane's
programming model. Addressing them individually:

* **Observers and event listeners**, which have long been considered an
  antipattern.
* **Classic component** functionality such as `classNames`, `classNameBindings`,
  `attributeBindings`, etc. will be unnecessary with Glimmer components.
* **Ember Data** provides computed properties which had to be manually wrapped
  in decorators. With the changes proposed in this RFC, however, they should
  continue to work without any additional changes. In fact, all computed
  property macros will.

Users who want these features will still be able to rely on addons such as Ember
Decorators, which will provide decorator support for them for the forseeable
future. Moving forward, this RFC breaks down into _computed properties_ and
_actions_.

### Computed Properties

As mentioned before, computed properties essentially _are_ decorators. However,
they are not spec compliant. Currently, `computed()` returns an instance of the
`ComputedProperty` class, which contains all of the meta information about the
decorated property. Native decorators, by contrast, are functions which receive
a descriptor and modify it as necessary.

Unfortunately, there's no way for us to know _ahead of time_ when a computed
property is going to be used as a native decorator in a native class, and when
it is going to be used in a classic class. Consider the following:

```js
class Person {
  @alias('prefix') title;
}
```

Really, what's going on there is _not_ that we are invoking the `@alias`
decorator with parameters. We are invoking a function which _returns_ a
decorator, so it desugars to:

```js
const aliasForPrefix = alias('prefix');

class Person {
  @aliasForPrefix title;
}
```

Therefore, the `alias` function _must_ itself return a decorator function.
However, this conflicts with usage in the classic programming model:

```js
const Person = EmberObject.extend({
  title: alias('prefix')
})
```

We just established that `alias` must return a decorator function, but here it
is with the exact same arguments, and it needs to return a `ComputedProperty`
instance. There is nothing we can branch on here - in both cases, `alias` only
receives the string `'prefix'`, so it has no context for how it will be used.

The native class piece of this puzzle is completely inflexible. A decorator must
be a function, there is no choice about it. However, the _Ember_ piece is _very_
flexible. The classic object model just needs a way to get the meta information
for the property when the class is being finalized. We can either assign the
meta information to the decorator function directly, or we can associate it via
a [`WeakMap`](https://developer.mozilla.org/en/docs/Web/JavaScript/Reference/Global_Objects/WeakMap).

The benefit of doing this is that the entire Ember ecosystem will get decorator
support with no extra work required. Because standard `computed` definitions
will work as decorators, existing macros will also work as decorators with no
changes to existing code.

All of Ember's built in computed macros in the `@ember/object/computed` module
will also become decorators with no extra work. However, the injection macros
will require slight updates as they use a subclass of the `ComputedProperty`
class. These updates should be relatively minor, and will follow the same
strategy as `computed()`.

#### Usage and API

The API for `computed` will remain mostly the same. The key differences will be:

1. The result of `computed` will be a decorator which can be applied directly to
   native _getters_, _setters_, and _class fields_.
2. The `ComputedPropertyConfig` (the getter/setter functions) argument provided
   to `computed` will now be optional when used as a decorator on a native
   getter or setter, and the native getter/setter will be used instead.

```ts
function computed(...args: (string | ComputedPropertyConfig)[]): PropertyDecorator;
```

The function signatures of all existing macros, including `inject` macros, would
change in the same way.

In general usage, these three definitions are equivalent:

```js
import EmberObject, { computed } from '@ember/object';

const Person = EmberObject.extend({
  fullName: computed('firstName', 'lastName', {
    get() {
      return `${this.firstName} ${this.lastName}`;
    }
  })
});

class Person {
  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}

class Person {
  @computed('firstName', 'lastName', {
    get() {
      return `${this.firstName} ${this.lastName}`;
    }
  })
  fullName;
}
```

That last example may seem confusing at first, but this is actually the same as
defining a computed property macro:

```js
import EmberObject, { computed } from '@ember/object';

function join(...dependentKeys) {
  return computed(...dependentKeys, {
    get() {
      return dependentKeys.map(key => this[key]).join(' ');
    }
  });
}

class Person {
  @join('firstName', 'lastName')
  fullName;
}
```

Notably, using `@computed` as a decorator _directly_, without parenthesis, will
not be supported. This is to prevent a parameter check on a critical path:

```js
class Person {
  @computed
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

The most common use case for this form in classic classes was to provide a new
instance of an object or array per instance of the class. This use case is
solved in native classes by class fields:

```js
const Person = EmberObject.extend({
  cache: computed(function() {
    return {};
  })
})

class Person {
  cache = {};
}
```

For other use cases, such as lazy evaluation and caching, it will still be
possible to call `@computed` with no arguments:

```js
class Person {
  @computed()
  get cache() {
    return {};
  };
}
```

#### Preventing Incoherent Usage

Making the `ComputedPropertyConfig` optional opens up lots of room for
accidents. A computed property without a getter or setter does not make sense,
nor does a computed propery with _two_ getters or setters. The new decorator
will assert at _decorator application_ time to ensure it is being used
correctly:

```js
import EmberObject, { computed } from '@ember/object';

// This will throw because the user attempted to define a CP without a getter
const Person = EmberObject.extend({
  fullName: computed()
});

// This will also throw because it is missing a getter
class Person {
  @computed('firstName', 'lastName')
  fullName;
}

// This will throw because a getter was already defined
class Person {
  @computed('firstName', 'lastName', {
    get() {
      return `${this.firstName} ${this.lastName}`;
    }
  })
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

Effectively, if `computed` is passed a `ComputedPropertyConfig`, it returns a
class field decorator. Otherwise, it returns an accessor (getter/setter)
decorator.

#### Property Modifiers

Almost all computed property modifiers have been deprecated at this point, but
they are still in use today and will still be available until Ember v4. As such,
their syntax needs to remain available and unchanged:

```js
import EmberObject, { computed } from '@ember/object';

const Person = EmberObject.extend({
  fullName: computed('firstName', {
    get() {
      return `${this.firstName} ${this.lastName}`;
    }
  }).readOnly().volatile().property('lastName') // a very strange combination
});
```

The decorator returned from `computed()` will need to have these chainable
methods available, and they will need to set the state of the decorator. This
should not be too difficult to accomplish.

Usage in native decorator syntax is a little bit trickier. In the current
proposal, only simple chaining is allowed in a decorator invocation. You may not
chain on the result of a function:

```js
import { computed } from '@ember/object';

const fullName = computed('firstName', 'lastName', function() {
  return `${this.firstName} ${this.lastName}`;
});

class Person {
  @computed('firstName', 'lastName').readOnly() // this is invalid JS decorator syntax
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  @fullName.readOnly() // this is valid because it's a simple chain
  otherFullName;
}
```

Luckily, there is one other form of invocation which is available - wrapping the
entire decorator expression in parenthesis:

```js
import { computed } from '@ember/object';

class Person {
  @(computed('firstName', 'lastName').readOnly()) // this is valid
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

This is clearly _not_ ideal for commonly used features, and while `volatile()`
and `property()` are not very well known, `readOnly()` is generally considered
best practice, and is used all over the place. However, it _is_ deprecated, and
in the future computed properties will be read only by default. Rather than
attempt to write a different API for decorators, this RFC proposes that we
accept the current syntax, and focus instead on the implementation of Svelte.
This will allow users to enable default read only CPs much sooner, and prevent
the need to use `readOnly()` at all.

##### A Tale of Two `readOnly`s

You may be wondering why we can't add more decorators to Ember to take the place
of these modifiers. The crux of the issue is the `readOnly()` modifier, and the
`readOnly()` macro. These share a name, and when macros become decorators as
well they will collide. The only difference would be the import paths, and this
would result in awkward renaming which would likely _not_ be conventional:

```js
import { readOnly, computed } from '@ember/object';
import { readOnly as readOnlyAlias } from '@ember/object/computed';

class Person {
  @readOnly
  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  @readOnlyAlias('fullName')
  otherFullName;
}
```

Ember Decorators made the decision to attempt renaming the macros themselves,
since `reads`, `readOnly`, and `oneWay` were all poorly named (arguably, `reads`
should have been `readOnly` since it is the more common use case) but this is
not a possibility in Ember, since this would cause collisions and confusion
within the macros namespace.

All of this would be to support a deprecated feature, which can still be used
(albeit, with a less-than-ideal syntax). Accepting the current syntax would also
prevent us from polluting the decorator namespace - we may want to use
`@readOnly` or `@volatile` in the future with tracked properties instead.

### Actions

Actions are special in Ember because they are namespaced within a class on the
`actions` object, despite being able to reference the class directly using
`this` when called, and otherwise behaving like standard methods. The reason for
this stems from the early days of Ember, when users would accidentally name an
action something that conflicted with existing lifecycle and event hooks (e.g.
`destroy`, or `click`). The `actions` namespace was added as a convenience to
prevent these collisions, and to separate them from the rest of the class body
organizationally.

This namespace is problematic for native classes for a number of reasons:

1. The namespace cannot be defined using class field syntax, since that would
   assign a copy of the object to every instance of the class, and there is no
   other native way to easily assign it.
2. Actions must inherit from the parent, which means that the `actions` object
   must have its prototype set to the parent class's `actions` object.
3. It is not possible to use `super` within non-class methods, meaning an
   alternative would have to be developed specifically for the namespace.

With these constraints, a decorator would be necessary to maintain the current
namespacing functionality. The options for such a decorator are limited. It
could:

1. Decorate a class field, placing the value on the class prototype and setting
   up inheritance/super functionality. This would necessarily result in either
   some amount of repetition in the decorator name and field name, or a
   redundant field name:

    ```js
    class Foo {
      @actions actions = {
        onClick() { /* ... */ }
      }
    }
    ```

2. Decorate the class itself with the actions object as a parameter. This would
   be awkward, since actions would be removed from the class definition:

    ```js
    @actions({
      onClick() { /* ... */ }
    })
    class Foo {}
    ```

3. Decorate the class itself, with the expectation that the `actions` class
   field exists and is an object. This leads to a disconnect between the
   decoration and the definition that is easy to miss, and could be
   counterintuitive to newcomers:

    ```js
    @actionHandler
    class Foo {
      actions = {
        onClick() { /* ... */ }
      }
    }
    ```

None of these options is ergonomic in the least. Instead, it is much cleaner and
easier to decorate method definitions that are directly on the class body:

```js
class Foo {
  @action
  onClick() { /* ... */ }
}
```

This would be implemented by creating the class's `actions` object and assigning
the method to it, setting up inheritance and such in the process. The decorator
leaves the method definition on the class, where it can be called like a normal
method would, including `super` functionality. This maintains compatibility with
the classic object model, while making the `actions` namespace an
_implementation_ detail rather than something users need to know about.

This does mean that actions can once again collide with actual lifecycle and
event hooks on the class, since they are no longer namespaced. The decorator
_could_ remove the method from the class definition entirely, but this would be
confusing for users not familiar with the old `actions` namespace - why does
this method disappear from the class? It would also break `super` functionality,
so it would not be ideal to do this.

The `@action` decorator _could_ warn users when it collides with a lifecycle
hook. However, hooks may vary from class type to class type, which presents a
design challenge. We could either:

1. Allow classes to specify lifecycle hooks, and throw whenever `@action`
   collides with a specified hook.
2. Only throw on hooks that are shared across all classes, such as `init` and
   `destroy`.
3. Do nothing, and leave it to user's to know which lifecycle hooks exist.

Specifying hooks for each class would be time consuming and could fall out of
sync with the implementations. Throwing on "universal" hooks only is
inconsistent, and could lead users to think that they are safe when they are
not. This RFC suggests that we choose option 3 for this reason. When the actions
namespace was introduced, lifecycle hooks like `destroy` and `click` were less
commonly known and used (event listeners were not uncommon). Most Ember users
know they exist now, and will be aware that implementing an action with the same
name is not recommended. In addition, eslint rules can be added to hint against
these collisions.

### Method Binding

`@action` will also bind the function to the class instance, allowing it to be
used in templates and elsewhere without having to be bound:

```js
export default class ButtonComponent extends Component {
  @action
  onClick() {
    // handle click
  }
}
```
```hbs
<button onclick={{this.onClick}}>Click me!</button>
```

#### Usage and API

The API for this new decorator would be much simpler than the computed API,
since it is only used as a decorator without parameters.

```ts
// Technically `action` is a function, but we can't type it transparently that way
const action: MethodDecorator;
```

Attempting to pass any parameters to the decorator, or to apply the decorator to
anything other than a class method, will throw an error.

## How we teach this

Teaching decorators is intrinsically tied to a wider shift in the Ember
programming model - the Ember Octane edition. From a teaching perspective, this
edition will be completely overhauling the guides and updating all of the best
practices as they stand. New users should see native class syntax with
decorators as the _default_, and should not ever have to write a classic class
or see an example for one.

With this RFC, the majority of existing examples in the Ember guides will be
updatable to native class syntax. The exception would be examples of classic
components, which would be addressed separately by updating the guides to
Glimmer Components (proposed in a separate RFC). Otherwise, all examples in the
guides should be updated.

### Updating and Interop

For existing users, or users who have to interact with classic code from a modern
context, it'll be important to have a reference for the classic object model.
The current section on the object model in the guides can be moved to a classic
section, and a section on updating should be added. Links to relevant codemods,
such as the
[ember-es6-class-codemod](https://github.com/scalvert/ember-es6-class-codemod),
should be included. This section should remain updated and included in the main
guides for as long as `EmberObject` is a part of Ember's public API.

## Acceptance Commitments

> This section serves to capture the various commitments accepting this RFC
> would entail.

* Adds support for decorators and class fields to Ember's public API. Transforms
  would be included out of the box as well.
* Allows computed properties to work as a native decorators on native classes.
* Adds the `@action` decorator for defining actions on native classes.

## Drawbacks

* The `ComputedProperty` class has long been considered intimate API. Even with
  recent changes as part of the native getter RFC to make it more private, these
  changes could still cause breakage.
* The strategy for converting `computed` to decorators has one major drawback,
  which is that decorator macros cannot easily be customized and will require a
  bit of boilerplate in some cases. For instance, currently in Ember Decorators
  it is possible to apply the `map` and `reduce` macros directly to a _method_,
  which becomes the method to map or reduce by:

  ```js
  @map('array')
  mappedArray() {}

  @map('array', function() {}) mappedArray;
  ```

  With this method, only the second form would be usable. Likewise, by default
  addons like Ember Data would need to write thin decorator wrappers around
  macros that may be called _without_ parameters, or which require key
  reflection:

  ```js
  @attr name; // This would not work OOTB
  @attr('string') name; // This would

  @belongsTo user; // This would not work OOTB
  @belongsTo('user') user; // This would
  ```

## Alternatives

### No Decorators

We could not have official Ember support for any aspects of the classic
programming model. This essentially means computed properties, since `@action`
and the injection helpers are still needed for the Octane model. This would
leave many users in limbo, unable to update to native class syntax fully because
it would mean rewriting large amounts of classes and components, and would make
libraries like `@ember-decorators` essential.

### Namespaced Decorators

We could include decorators as a separate package, such as `@ember/decorators`.
This is not ideal as it would force users to remember more import paths, and it
would make eventual deprecation of the classic form much more difficult. It would
also mean that the wider ecosystem would have to do much more work to adopt
decorator syntax.

### Full Compatibility Decorators

We could include decorators for the remaining classic features: Observers, event
listeners, and classic components. These would add extra weight, and may
encourage users to continue using these features, which would not be ideal.

Instead, we can recommend that users wanting to update to native class syntax
use external packages that implement these features, such as
`@ember-decorators`. The native class codemod will detect and automatically
include these packages if they are necessary.

### Default Read Only Decorator

In this proposal, computeds used as decorators match the semantics of computeds
used in classic classes exactly. This is true even in the unfortunate case of
computed overridability:

```js
class Person {
  firstName = 'Stefan';
  lastName = 'Penner';

  @computed('firstName', 'lastName')
  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}

let person = new Person;

person.fullName; // 'Tom Dale'

person.set('firstName', 'Kris');
person.set('lastName', 'Selden');

person.set('fullName', 'Melanie Sumner'); // overrides the setter

person.fullName; // 'Melanie Sumner'
```

We could, instead, make decorators apply computeds as _readOnly by default_,
since this is a new usage of them and not a breaking change. This would require
us to add a new `overridable()` modifier to opt-out of readOnly behavior by
default, for backwards compatibility, and would add a fair amount of complexity
to codemods. Either way, this behavior will be the default in Ember v4, and
deprecations will begin appearing when this happens soon.

### Allow `@action` to Rename Actions

If the namespace collisions caused by actions becoming standard methods are
difficult to refactor around or codemod, we can consider allowing the `@action`
helper to receive an alternative action name:

```js
export default class ButtonComponent extends GlimmerComponent {
  @action('destroy')
  destroyAction() {
    // handle click
  }
}
```
```hbs
<button {{action 'destroy'}}>Click me!</button>
```

This could be added later by a followup RFC, so it is not part of this proposal.
Ideally it won't be necessary.

## Unresolved questions

As stated in the introduction, this RFC is being made with the assumption that
decorators will be moved to stage 3 before this RFC is actually implemented. If
they are _not_ moved to stage 3, we will have to decide if decorators should be
supported in while they are in stage 2.

If they are supported while in stage 2, there are some additional questions:

* Should stage 2 transforms continue to be supported after decorators move to
  stage 3? Would removing stage 2 support require a major version bump?
* Should Typescript's stage 1-like decorators be supported, since Typescript
  will not implement new decorator transforms until they reach stage 3?

## Appendix A

This appendix contains the list of affected APIs and new APIs for quick reference.

| `@ember/controller` |
|---------------------|
| `inject`            |

| `@ember/object` |
|-----------------|
| `computed`      |
| _`action`_      |

| `@ember/object/computed` |
|--------------------------|
| `alias`                  |
| `and`                    |
| `bool`                   |
| `collect`                |
| `deprecatingAlias`       |
| `empty`                  |
| `equal`                  |
| `filter`                 |
| `filterBy`               |
| `gt`                     |
| `gte`                    |
| `intersect`              |
| `lt`                     |
| `lte`                    |
| `map`                    |
| `mapBy`                  |
| `match`                  |
| `max`                    |
| `min`                    |
| `none`                   |
| `not`                    |
| `notEmpty`               |
| `oneWay`                 |
| `or`                     |
| `readOnly`               |
| `reads`                  |
| `setDiff`                |
| `sort`                   |
| `sum`                    |
| `union`                  |
| `uniq`                   |
| `uniqBy`                 |

| `@ember/service` |
|------------------|
| `inject`         |

## Appendix B

This appendix contains a full list of changed APIs and their old and new signatures.

### **`@ember/controller`**

* `inject`
  ```ts
  // old
  function inject(): ComputedProperty<Controller>;
  function inject<K extends keyof ControllerRegistry>(
      name: K
  ): ComputedProperty<ControllerRegistry[K]>;

  // new
  function inject(): PropertyDecorator;
  function inject<K extends keyof ControllerRegistry>(
      name: K
  ): PropertyDecorator;
  ```

### **`@ember/object`**

* `computed`
  ```ts
  // old
  function computed(...args: (string | ComputedPropertyConfig<T>)[]): ComputedProperty<T>;

  // new
  function computed(...args: (string | ComputedPropertyConfig)[]): PropertyDecorator;
  ```

* `action`
  ```ts
  // old
  // N/A

  // new
  const action: MethodDecorator;
  ```

### **`@ember/object/computed`

* `alias`
  ```ts
  // old
  function alias(dependentKey: string): ComputedProperty<any>;

  // new
  function alias(dependentKey: string): PropertyDecorator;
  ```

* `and`
  ```ts
  // old
  function and(...dependentKeys: string[]): ComputedProperty<boolean>;

  // new
  function and(...dependentKeys: string[]): PropertyDecorator;
  ```

* `bool`
  ```ts
  // old
  function bool(dependentKey: string): ComputedProperty<boolean>;

  // new
  function bool(dependentKey: string): PropertyDecorator;
  ```

* `collect`
  ```ts
  // old
  function collect(...dependentKeys: string[]): ComputedProperty<any[]>;

  // new
  function collect(...dependentKeys: string[]): PropertyDecorator;
  ```

* `deprecatingAlias`
  ```ts
  // old
  function deprecatingAlias(
    dependentKey: string,
    options: { id: string; until: string }
  ): ComputedProperty<any>;

  // new
  function deprecatingAlias(
    dependentKey: string,
    options: { id: string; until: string }
  ): PropertyDecorator;
  ```

* `empty`
  ```ts
  // old
  function empty(dependentKey: string): ComputedProperty<boolean>;

  // new
  function empty(dependentKey: string): PropertyDecorator;
  ```

* `equal`
  ```ts
  // old
  function equal(dependentKey: string, value: any): ComputedProperty<boolean>;

  // new
  function equal(dependentKey: string, value: any): PropertyDecorator;
  ```

* `filter`
  ```ts
  // old
  function filter(
    dependentKey: string,
    callback: (value: any, index: number, array: any[]) => boolean
  ): ComputedProperty<any[]>;

  // new
  function filter(
    dependentKey: string,
    callback: (value: any, index: number, array: any[]) => boolean
  ): PropertyDecorator;
  ```

* `filterBy`
  ```ts
  // old
  function filterBy(
    dependentKey: string,
    propertyKey: string,
    value?: any
  ): ComputedProperty<any[]>;

  // new
  function filterBy(
    dependentKey: string,
    propertyKey: string,
    value?: any
  ): PropertyDecorator;
  ```

* `gt`
  ```ts
  // old
  function gt(dependentKey: string, value: number): ComputedProperty<boolean>;

  // new
  function gt(dependentKey: string, value: number): PropertyDecorator;
  ```

* `gte`
  ```ts
  // old
  function gte(dependentKey: string, value: number): ComputedProperty<boolean>;

  // new
  function gte(dependentKey: string, value: number): PropertyDecorator;
  ```

* `intersect`
  ```ts
  // old
  function intersect(...propertyKeys: string[]): ComputedProperty<any[]>;

  // new
  function intersect(...propertyKeys: string[]): PropertyDecorator;
  ```

* `lt`
  ```ts
  // old
  function lt(dependentKey: string, value: number): ComputedProperty<boolean>;

  // new
  function lt(dependentKey: string, value: number): PropertyDecorator;
  ```

* `lte`
  ```ts
  // old
  function lte(dependentKey: string, value: number): ComputedProperty<boolean>;

  // new
  function lte(dependentKey: string, value: number): PropertyDecorator;
  ```

* `map`
  ```ts
  // old
  function map<U>(
    dependentKey: string,
    callback: (value: any, index: number, array: any[]) => U
  ): ComputedProperty<U[]>;

  // new
  function map<U>(
    dependentKey: string,
    callback: (value: any, index: number, array: any[]) => U
  ): PropertyDecorator;
  ```

* `mapBy`
  ```ts
  // old
  function mapBy(dependentKey: string, propertyKey: string): ComputedProperty<any[]>;

  // new
  function mapBy(dependentKey: string, propertyKey: string): PropertyDecorator;
  ```

* `match`
  ```ts
  // old
  function match(dependentKey: string, regexp: RegExp): ComputedProperty<boolean>;

  // new
  function match(dependentKey: string, regexp: RegExp): PropertyDecorator;
  ```

* `max`
  ```ts
  // old
  function max(dependentKey: string): ComputedProperty<number>;

  // new
  function max(dependentKey: string): PropertyDecorator;
  ```

* `min`
  ```ts
  // old
  function min(dependentKey: string): ComputedProperty<number>;

  // new
  function min(dependentKey: string): PropertyDecorator;
  ```

* `none`
  ```ts
  // old
  function none(dependentKey: string): ComputedProperty<boolean>;

  // new
  function none(dependentKey: string): PropertyDecorator;
  ```

* `not`
  ```ts
  // old
  function not(dependentKey: string): ComputedProperty<boolean>;

  // new
  function not(dependentKey: string): PropertyDecorator;
  ```

* `notEmpty`
  ```ts
  // old
  function notEmpty(dependentKey: string): ComputedProperty<boolean>;

  // new
  function notEmpty(dependentKey: string): PropertyDecorator;
  ```

* `oneWay`
  ```ts
  // old
  function oneWay(dependentKey: string): ComputedProperty<any>;

  // new
  function oneWay(dependentKey: string): PropertyDecorator;
  ```

* `or`
  ```ts
  // old
  function or(...dependentKeys: string[]): ComputedProperty<boolean>;

  // new
  function or(...dependentKeys: string[]): PropertyDecorator;
  ```

* `readOnly`
  ```ts
  // old
  function readOnly(dependentKey: string): ComputedProperty<any>;

  // new
  function readOnly(dependentKey: string): PropertyDecorator;
  ```

* `reads`
  ```ts
  // old
  function reads(dependentKey: string): ComputedProperty<any>;

  // new
  function reads(dependentKey: string): PropertyDecorator;
  ```

* `setDiff`
  ```ts
  // old
  function setDiff(setAProperty: string, setBProperty: string): ComputedProperty<any[]>;

  // new
  function setDiff(setAProperty: string, setBProperty: string): PropertyDecorator;
  ```

* `sort`
  ```ts
  // old
  function sort(
    itemsKey: string,
    sortDefinition: string | ((itemA: any, itemB: any) => number)
  ): ComputedProperty<any[]>;

  // new
  function sort(
    itemsKey: string,
    sortDefinition: string | ((itemA: any, itemB: any) => number)
  ): PropertyDecorator;
  ```

* `sum`
  ```ts
  // old
  function sum(dependentKey: string): ComputedProperty<number>;

  // new
  function sum(dependentKey: string): PropertyDecorator;
  ```

* `union`
  ```ts
  // old
  function union(...propertyKeys: string[]): ComputedProperty<any[]>;

  // new
  function union(...propertyKeys: string[]): PropertyDecorator
  ```

* `uniq`
  ```ts
  // old
  function uniq(propertyKey: string): ComputedProperty<any[]>;

  // new
  function uniq(propertyKey: string): PropertyDecorator;
  ```

* `uniqBy`
  ```ts
  // old
  function uniqBy(dependentKey: string, propertyKey: string): ComputedProperty<any[]>;

  // new
  function uniqBy(dependentKey: string, propertyKey: string): PropertyDecorator;
  ```

### **`@ember/service`**

* `inject`
  ```ts
  // old
  function inject(): ComputedProperty<Service>;
  function inject<K extends keyof ServiceRegistry>(
      name: K
  ): ComputedProperty<ServiceRegistry[K]>;

  // new
  function inject(): PropertyDecorator;
  function inject<K extends keyof ServiceRegistry>(
      name: K
  ): PropertyDecorator;
  ```
