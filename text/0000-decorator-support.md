- Start Date: 2019-02-06
- Relevant Team(s): Ember.js, Ember Data, Ember CLI, Learning, Steering
- RFC PR:
- Tracking: (leave this empty)

# Decorator Support

## Summary

Move forward with the Decorators RFC, and support decorators while they remain
in stage 2 and move forward through the TC39 process.

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
the case. This is also the feedback that the core team received in the January
TC39 meeting. _Crucially_, all parties were in agreement about the _invocation_
syntax of decorators:

```js
class Person {
  @decorate name;
}
```

This is the most stable part of the spec, and the one that average developers
will come into contact with most. It means that changes to the spec will likely
affect the _application_ of decorators, which will in turn affect library and
framework maintainers, but **not** end users in most cases. This places the
burden of dealing with the spec changes on the Ember.js core team, and addon
authors who wish to adopt decorators while they are in stage 2.

As such, adopting decorators now should present a minimal amount of risk to
Ember and its users, and while it is not ideal, it allows us to continue moving
forward and innovating as a framework.

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
users want to write Glimmer-like component's with classic syntax.

### Decorator Versioning and Support

Because the decorators spec has changed while remaining in a stage 2, it's no
longer enough to refer to "stage 2" as a version. Since Babel will be providing
the transforms, we will use their versioning system. [This is currently in
development](https://github.com/babel/babel/pull/9360), but looks like it will
be based on the TC39 meeting that the spec was presented at. For instance, the
current transforms available in Babel are referred to as `nov-2018`.

Ember will begin by supporting the _latest_ version of the decorators transform
provided by Babel. Currently this is the `nov-2018` version, but by the time
decorators are released this may change.

The decorators version will be configurable on a _per-app/engine/addon_ basis.
This will allow the ecosystem to adopt decorators without fears that if an
application updates, it could break their decorators. If no version is
specified, the transform will default to the latest version supported by Ember
at that time. In many cases the only decorators in use will be Ember's, so this
allows the ecosystem to upgrade without additional maintenance costs. The exact
method for specifying the version will be determined during implementation, but
will likely be a configuration option or addon.

When new transforms are released, Ember will continue to support older versions
of the transforms alongside the new ones until _at least_ the next LTS release,
with deprecation warnings for any users of the older transforms. This will mean
shipping multiple different versions of decorators at once, and providing a way
for apps/addons to import the correct version. Additionally, Ember will attempt
to provide addon authors with tools that allow _them_ to do this as well, so
they can support multiple versions of transforms easily and transparently. The
exact API for these tools is out of scope for this RFC, and will likely be
dependent on their implementation.

## How we teach this

As mentioned in the motivation section, the _invocation_ side of decorators
seems to be much more stable currently, so we ideally will not have to teach end
consumers too much about the details of this system. We should definitely
provide a disclaimer that outlines how our support structure works, and how
users can lock their version if they are using custom decorators. This should be
included _early_ in the guides.

We should also provide thorough documentation for any tooling provided to help
addon authors write their own decorators. This will be a bit more of a moving
target, since this RFC does not define exact APIs and they will not be a
priority until Ember itself prepares to support multiple versions of the
transforms.

## Drawbacks

The fact that decorators did not move to stage 3 signals that there may be
additional changes to the spec in the near future. Adopting them now could cause
more churn than it's worth.

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
