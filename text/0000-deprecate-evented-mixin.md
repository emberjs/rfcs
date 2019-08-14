- Start Date: 2019-08-14
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Deprecate Evented Mixin

## Summary

Deprecation support for the `Evented` Mixin.

## Motivation

Mixins in Ember can only be used when inheriting from an EmberObject, and with the introduction of native javascript classes the syntax to use them looks a little odd.

```
class MyClass extends EmberObject.extend(Evented) { }
```

Furthermore, the Evented Mixin is just a wrapper around a set of functions exposed from `@ember/object/events`.

Deprecating the Evented Mixin, should make it easier for applications reduce the dependency on EmberObject and make it easier to switch to native class syntax.

## Transition Path

As mentioned in the motivation, the `@ember/objects/events` package already exposes the functionality of the Evented mixin

There is a one to one mapping for each function in the Evented mixin as listed below:

On:
`eventedObj.on(eventName, target, method)` -> `addListener(eventedObj, eventName, target, method)`

One:
`eventedObj.one(eventName, target, method)` -> `addListener(eventedObj, eventName, target, method, true)`

Off:
`eventedObj.off(eventName, target, method)` -> `removeListener(eventedObj, eventName, target, method)`

Trigger:
`eventedObj.trigger(eventName, param1, param2, ..., paramN)` -> `sendEvent(eventedObj, eventName, [param1, param2, ..., paramN])`

Has:
`eventedObj.has(eventName)` -> `hasListeners(eventedObj, eventName)`

Currently `hasListeners` is not a function that can be imported from `@ember/object/events`. `hasListeners` would need to be exported for this deprecation to be possible.

To assist with this transition, it should be possible to provide a codemod that updates the application to make use of the events functions/

## How We Teach This

The guides currently make no mention of the Evented mixin. Thus, the best place to teach this would be in the API documentation for the Evented mixin.

The Evented page could be updated to direct users to make use of the `@ember/object/events` package instead.
For each function in the Evented Mixin (`has`, `off`, `on`, `one`, `trigger`) an example could be shown of how to write the code using the functions exposed in the `@ember/object/events` package, similar to above.

It should be possible to provide a codemod, to assist with this transition.

## Drawbacks

This RFC introduces churn in existing apps that make use of the Evented Mixin.

## Alternatives

Do not deprecate the Evented Mixin

## Unresolved questions

None
