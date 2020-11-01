---
Start Date: 2019-02-06
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/440
Tracking: https://github.com/emberjs/rfc-tracking/issues/41

---

# Decorator Support

## Summary

Move forward with the Decorators RFC via stage 1 decorators, and await a stable
future decorators proposal.

## Motivation

The recently merged [Decorators
RFC](https://github.com/emberjs/rfcs/blob/master/text/0408-decorators.md)
contained an explicit condition: That it was premised on decorators moving from
stage 2 in the TC39 process to stage 3. If this were to not happen, the original
RFC stipulates that a followup (this RFC) should be created to clarify whether
or not we should move forward with decorators while they remain in stage 2, and
how we would do so.

This RFC proposes that we do so. As discussed in the original, the decorator
pattern has been in use in Ember since the beginning, and there is no easy path
forward to native class syntax without a native equivalent to "classic"
decorators.

[Stage 2 in TC39](https://tc39.github.io/process-document/) signals that the
committee expects this feature to be included in the language in the future, and
is working out the details and semantics. The fact that decorators _remain_ in
stage 2, and have not been rejected, signals that they still expect this to be
the case. However, it is clear based on the initial work following the January
TC39 meeting that [the proposal could change
significantly](https://github.com/tc39/proposal-decorators/pull/250) between now
and stage 3.

Parts of the proposal, such as how decorators are invoked, seem solid based on
the feedback we received at the January TC39 meeting, and based on the draft of
the new spec. The definition of decorators is the most likely thing to change.
As such, user code should be minimally affected by any changes, and most changes
should be codemod-able. This reduces the risk of adopting decorators now, since
code _written_ with decorators shouldn't need to change that much.

[The current recommendation from the authors of the
spec](https://github.com/tc39/proposal-decorators/tree/static#how-should-i-use-decorators-in-transpilers-today)
is to use the stage 1 decorators proposal until the next iteration is ready.
Stage 1 decorators are very stable, and used by a large community of developers
outside of Ember. By standardizing on this version of the spec, we'll be able to
gain the benefits of using a shared solution with the wider ecosystem while we
wait for the next iteration.

## Detailed design

### Classic Classes

The first thing to note is that Ember.js will continue to work with classic
classes and classic class syntax for the forseeable future, as a way for users
to opt-out if they don't want to take the risk of using native classes. This
includes:

- Classic Classes
- Classic Components
- Mixins
- Computed Properties
- Observers
- Tracked Properties
- Injections
- All existing classes defined with classic class syntax

Notably, `GlimmerComponent` will _not_ support classic class syntax, due to its
constraint of having to support both Ember.js _and_ Glimmer.js, where the
classic class model is not available. However, creating an API compatible
classic class version using Ember's component manager API should be possible if
users want to write Glimmer-like components with classic syntax.

### Decorator Semantics

Ember will support Babel's stage 1 decorator transforms. Since Babel and
TypeScript's decorators overlap so much, most TypeScript decorators should also
work in Ember apps. However, due to subtle differences in the way Babel and
TypeScript's decorators work, we can't support them both in _all_ cases. In
cases where there are nuanced differences, Babel's transforms will be considered
the canonical source of truth. For `ember-cli-typescript` users this shouldn't
be an issue, since they can use Babel's transforms in the latest versions of
`ember-cli-typescript`.

#### Class Field Assigment Order

Ember also does _not_ guarantee a particular ordering for the assignment of
decorated fields in order to support users who want to use _native_ class fields
when they are available. All that Ember guarantees is that by the time the
`constructor`'s `super` method completes (or the constructor code begins, if the
class is not extending), all class fields will be assigned.

Assigning class fields based on the values of other class fields is somewhat of
an anti-pattern as is, so this would reinforce discouraging problematic
patterns. To be clear:

```js
class MyComponent extends Component {
  // This is ok, because the component instance exists
  // for all class fields.
  stateManager = new StateManager(this);

  @service time;

  // This is ok, because the `time` is an injection,
  // and always available for all class fields.
  createdAt = this.time.now();

  // This is problematic in general, not just with decorators, because
  // it relies on the order of class field assignment within this class.
  // It is a refactoring hazard, and should likely be done in the
  // constructor instead.
  registry = new Registry();
  container = new Container(this.registry);
}
```

Ember will attempt to provide a lint rule which can handle this level of nuance
and prevent frustration when users copy and paste code around the class body.

#### Class Field Assignment Semantics

Currently, the Babel stage 1 transforms require `loose` mode for class fields
which causes them to be assigned directly rather than using
`Object.defineProperty`, which is not inline with the class fields spec. The
biggest difference in behavior is when a child class attempts to override a
getter/setter on the parent class:

```js
class Foo {
  get baz() {
    return this._baz;
  }

  set baz(value) {
    this._baz = value;
  }
}

class Bar extends Foo {
  baz = 123;
}
```

In strict mode, `baz` in the child class would completely override the
getter/setter, and they would not exist/be usable on an instance of `Bar`
(except through calls to `super`). In loose mode, `baz` in the child class would
go through the getter/setter, and be assigned to `_baz` on the instance.

In order to mitigate this, Ember will provide an assertion in development builds
via a babel transform for fields that are assigned in a non-spec compliant way
that throws in this scenario. This will prevent users from accidentally writing
code that will be hard to migrate forward to class fields in the future.

### Decorator Support Timeframe

Stage 1 decorators will be considered a first class Ember API. They will be
supported until:

1. A new RFC has been made to introduce a new decorator API as a parallel API to
   the stage 1 decoraters.
2. A deprecation RFC has been created for Stage 1 decorators.

They will follow the same deprecation flow, and same SemVer requirements, as any
other Ember feature.

## How we teach this

There are a few different important aspects of this that should be taught:

1. **Class Field and Decorator semantics**, specifically around the ordering of
   class fields. While any step to change a user's class field ordering would
   likely be based on their target browsers and configuration, and would be
   unlikely to change in a patch or minor version release, users writing code
   that could be difficult to upgrade or dependent on a particular version of
   the class field/decorator transforms is a failure case we want to avoid.
2. **Support timeframe and alternatives.** Users should be aware that this will
   be a feature that will _likely_ change in the future, if only subtly. We
   should highlight that there _is_ an alternative, classic class syntax, and
   that it will still be fully supported. Ultimately, this point is about
   setting expectations - that a year or so from now, when decorators are truly
   final, there will be another shift.
3. **Custom Decorators.** We should document how users can write their own
   decorators, but also caution against behaviors that will be difficult to
   replicate in Stage 3 decorators when they are ready. This will _necessarily_
   be a moving target, since stage 3 decorators are not yet finished, but we can
   try our best to recommend against usages that could be problematic.

   We should also warn that decorator definition code will almost definitely
   have to be rewritten for stage 3. This should be very clear, so that users
   understand that adopting custom decorators in their apps means taking on more
   tech debt than just using official Ember decorators, and decorators provided
   by addons.

## Drawbacks

- The fact that decorators did not move to stage 3 signals that there may be
  additional changes to the spec in the near future. Adopting them now could
  cause more churn than it's worth.

- By adopting decorators now, we are essentially taking on some amount of debt
  that we know will have to be repaid in the future as a community. This is less
  than ideal, since we know that standardization is coming, it's just a matter
  of when. This is also the situation we've been in for quite some time already,
  and with the latest turn of events, it seems unlikely that decorators will be
  fully standardized within the year.

  We can continue to wait, but there is no deadline on the design process, and
  we could be stuck here indefinitely. This move is pragmatic in that it
  unblocks us for now, and moves us toward using modern syntax that will be
  compatible with stage 3 decorators - it allows us to begin unwinding other
  layers of technical debt. It also represents minimal risk and debt to _users_
  of decorators, since the invocation style will likely be the same.

## Alternatives

- We could continue to rely on unofficial addons such as `ember-decorators`,
  which allow users to opt-in to using decorators. However, these libaries have
  limitations that first class decorators will not have, and they don't allow us
  to update the official guides to use them.

- We could create an official decorators addon. However, this means that
  decorators would be available at a different import path, meaning that any
  code which seeks to work with _both_ classic classes and native classes would
  have to be written twice. This would be very difficult for applications that
  are mid-transition, and even more difficult throughout the ecosystem for addon
  authors.
