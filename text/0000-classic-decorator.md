- Start Date: 2019-03-14
- Relevant Team(s): Ember.js, Ember Data, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/468
- Tracking:

# `@classic` Decorator

## Summary

Add a set of warnings for users who adopt native class syntax with
`EmberObject` base classes, and a `@classic` decorator which can be used to
disable these warnings on a per-class basis.

## Motivation

As we've added native class support to Ember and moved forward with Ember
Octane, it's become increasingly clear that there are still a number of strange
behaviors, edge-cases, and other issues with adopting native class syntax and
extending from `EmberObject`. These issues include:

- The timings of `init` and `constructor`, and how the two can interact through
  a class hierachy in a way that causes confusing failures.
- The differences between class fields and properties, and how class fields can
  override properties even in a subclass.
- The use of Mixins, which lack a native class alternative, and can force users
  to interop and have even more failure cases.

Despite these edge cases, the community has made progress using native class
syntax, and it is absolutely necessary for some users (`ember-cli-typescript`
users in particular). The ability to transition progressively to a new syntax is
also very valuable, and allowing users to adopt native class syntax _without_
completely rewriting their classes means that they can upgrade one step at a
time.

However, as we begin to move into a world where more and more classes _don't_
extend from `EmberObject` (starting with `GlimmerComponent`), these edge cases
will become more common and confusing. It won't be enough to rely on simple
rules of thumb like "always use `init`" because in some cases, there won't _be_
an `init` method. It'll also become a question of "which class am I extending?
`GlimmerComponent`? `EmberComponent`?

This RFC proposes adding a `@classic` decorator which can be used on classes
that still use `EmberObject` APIs:

```js
import classic from 'ember-classic-decorator';

@classic
export default class ApplicationController extends Controller {
  init() {
    super.init(...arguments);
    // setup the controller class
  }
}
```

This decorator will:

1. Provide a visual hint to users that they class they are working on uses
   `EmberObject` specific functionality, and to be aware that they are not
   working with "just" native classes.
2. Provide a hint to _linters_ so that they can use a different set of rules for
   "classic" classes, such as always use `init`, etc. and native classes, such
   as to never use `.extend()` or mixins.
3. Disable a number of warnings that will alert users when they have encountered
   an edge case, such as using `init` in a parent class and `constructor` in a
   child class.

This should allow users who wish to adopt native class syntax to do so
incrementally, one change at a time, with as much safety as possible.

## Detailed design

There are roughly two categories of base classes in Ember:

1. Base classes which do not have alternatives, _and_ use a very minimal amount
   of `EmberObject` APIs
   - `Route`
   - `Controller`
   - `Service`
   - `Helper`
2. Base classes which have modern alternatives, _and_ use a much larger amount
   of `EmberObject` APIs:
   - `Component`, which can be converted to `GlimmerComponent`
   - Utility classes, which can be converted plain native classes that do not
     extend `EmberObject`

`@classic` will work slightly differently for these two categories. We'll call
them _transitionable_ and _non-transitionable_ respectively.

### Transitionable Base Classes

Transitionable base classes can be converted to native classes safely _without_
using any `EmberObject` APIs. Once converted, we can safely lint against using
any `EmberObject` APIs, preventing any future confusion. As such, it will only
be necessary to apply the `@classic` decorator for as long as any of the user's
own classes use `EmberObject` APIs. These include:

- `extend`
- `reopen` and `reopenClass`
- `init`
- `destroy`
- `proto`
- `detect`
- `get`/`set`

Along with any other extra methods or APIs that exist on `CoreObject` and
`EmberObject`. Methods and APIs that exist on the class itself, such as
`transitionTo` on Routes, or `send` and `sendAction` on Controllers, are _not_
considered `EmberObject` APIs, since they could be implemented purely in native
classes, and aren't tied to the way `EmberObject` does inheritance.

If a class, or any of its parent classes, uses one of these methods _without_
being decorated with `@classic`, a lint rule will log a warning to let users
know that they are using classic object APIs, and they should either refactor
away from using those APIs, or use the `@classic` decorator to opt-out of
warnings.

The `@classic` decorator itself will brand the class it is applied to, disabling
warnings on that instance of the class.

### Non-transitionable Base Classes

Non-transitionable base classes _cannot_ be converted to native classes without
relying on behaviors that are specific to `EmberObject`. Classic components, for
instance, rely deeply on the way that `EmberObject.create` sets up its instance,
and any utility class that extends from `EmberObject` _must_ use `create` to
create instances. While this is also true of `Route`s and `Service`s, the
important distinction is that _users_ don't have to ever use `create` in their
own code, or are not passed any arguments other than injections, so for those
classes the usage of `create` is an implementation detail.

These classes will _always_ warn users if they are not decorated with
`@classic`. In order to transition away the decorator, they must be converted
into newer base classes, such as `GlimmerComponent`, or away from base classes
entirely.

### Implementation

The decorator and the warnings would be added by an official Ember addon,
`ember-classic-decorator`. This would allow us to keep the implementation
details separate from the main Ember codebase, especially specifics like babel
transforms for stripping out the `@classic` decorator in production builds.

The `@classic` decorator should only be applied in DEBUG mode, and should be
stripped entirely from production builds. Other than that, the details of the
exact implementation is left up to the champions, though it should likely use
symbols to brand classes marked with the decorator.

## How we teach this

In the Working with JavaScript section of the guides, when we discuss both
native class syntax and classic class syntax, we should also provide a breakdown
of using native classes with objects that implement or use classic class APIs.
We can explain the usage of the `@classic` decorator here, along with clear
examples for how it should be applied, and what will warnings without it.

The warnings themselve should also link to this guide page, with a clear
explanation for _why_ the warning was triggered, and a link to a section on the
page that covers the specific API, and how to convert that API to use native
class syntax. In the case of Glimmer components and utility classes, the
sections should point to the more detailed Octane Edition guide, which covers
the various differences between classic and Glimmer components and classic and
native classes, and how to convert the two.

## Drawbacks

Adding the `@classic` decorator could encourage more usage of native classes
with `EmberObject`, which in turn could lead to even more confusion as users
attempt to navigate the edge cases in differences between native and classic
syntax. If the warnings provided by `@classic` are not strong enough, it could
result in a frustrating developer experience when the differences between the
two cause problems and failures.

## Alternatives

This RFC is taking the stance that:

1. Adopting native class syntax with `EmberObject` is valuable enough to a
   significant number of users that we should support it, and recommend it in
   some cases (such as for transitionable classes)
2. There are enough caveats with doing this that we must warn users of them in
   order to support it.

We could alternatively _not_ recommend using native class syntax with any class
that extends `EmberObject`, even Route/Controller/Service and other
transitionable classes, and wait until we have a pure native alternative to
transition to.

We could also not warn, if we believe the caveats are actually not that
problematic, and that users can figure out the details on their own without any
hinting.

### Built-in Instead of Addon

This RFC currently proposes adding `@classic` as a default addon, which will
allow us to keep its implementation separate from the main Ember codebase. This
could lead to users dropping it or not adding it to existing codebases, which
would in turn lead to users encountering failures without any warnings.
