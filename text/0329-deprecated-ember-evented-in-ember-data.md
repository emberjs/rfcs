---
Start Date: 2018-05-01
Relevant Team(s): Ember Data
RFC PR: https://github.com/emberjs/rfcs/pull/329
Tracking: https://github.com/emberjs/rfc-tracking/issues/21

---

# Deprecate Usage of Ember Evented in Ember Data

## Summary

`Ember.Evented` functionality on `DS.Model`, `DS.ManyArray`,
`DS.Errors`, `DS.RecordArray`, and `DS.PromiseManyArray` will be
deprecated and eventually removed in a future release. This includes
the following methods from the
[Ember.Evented](https://www.emberjs.com/api/ember/2.15/classes/Ember.Evented/methods/on?anchor=off)
class: `has`, `off`, `on`, `one`, and `trigger`. Additionally the
following lifecycle methods on `DS.Model` will also be deprecated:
`becameError`, `becameInvalid`, `didCreate`, `didDelete`, `didLoad`,
`didUpdate`, `ready`, `rolledBack`.

## Motivation

The use of `Ember.Evented` is mostly a legacy from the pre 1.0 days of
Ember Data when events were a core part of the Ember Data programming
model. Today there are better ways to do everything that once needed
events. Removing the usage of the `Ember.Evented` mixin will make it
easier for Ember Data to eventually transition to using native ES2015
JavaScript classes and will reduce the surface area of APIs that Ember
Data must support in the long term.

## Detailed design

`Ember.Evented` mixin will be scheduled to be removed from the
following classes in a future Ember Data release: `DS.Model`,
`DS.ManyArray`, `DS.Errors`, `DS.RecordArray`, and
`DS.PromiseManyArray`.

The `has`, `off`, `on`, `one`, and `trigger` methods will be trigger a
deprecation warning when called and will be completly in a future
Ember Data release.

A special deprecation will be logged when users of a
`DS.adapterPopulatedRecordArray` attempt to listen to the `didLoad`
event. This depecations will prompt users to use a computed property
instead of the `didLoad` event.

`DS.Model` will also recieve deprecation warnings when a model is
defined with the following methods: `becameError`, `becameInvalid`,
`didCreate`, `didDelete`, `didLoad`, `didUpdate`, `ready`,
`rolledBack`.

When a model is instantiated for the first time with any of these
methods a deprecation warning will be logged notifiying the user that
this method will be deprecated and the user should use an computed or
overide the model's init method instead.

## How we teach this

Today we do not teach the use of any of the Ember Data lifecycle
events in the guides. They are referenced in the API docs but they
will be updated to mark the APIs as deprecated and show alternative
examples of how to achieve the same functionality using a non event
pattern.

The deprecation guide app will be updated with examples showing how to
migrate away from an evented pattern to using a computed or imperative
method to achieve the same results.

## Drawbacks

The drawback to making this change is existing code that takes
advantage of the Ember Data lifecycle events will need to be updated
to use a different pattern.

## Alternatives

We could leave the `Ember.Evented` mixin on all of the Ember Data
objects that currently support it and continue to support this
interface for the foreseeable future. However, Ember Data itself
doesn't require these events internally. There is only one place in
the `DS.Error` code that takes advantage of the `Ember.Evented` system
and that code can be easilly re-written to avoid `Ember.Evented` APIs.

## Unresolved questions

None
