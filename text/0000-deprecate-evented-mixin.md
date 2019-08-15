- Start Date: 2019-08-15
- Relevant Team(s): Ember.js, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/528
- Tracking: (leave this empty)

# Deprecate Evented Mixin

## Summary

Deprecation support for the `Evented` Mixin.

## Motivation

Mixins in Ember can only be used when inheriting from an EmberObject, and with the introduction of native javascript classes the syntax to use them looks a little odd.

```
class MyClass extends EmberObject.extend(Evented) { }
```

Furthermore, the Evented Mixin is just a wrapper around a set of functions from the `@ember/object/events` package.

Deprecating the Evented Mixin, should make it easier for applications reduce the dependency on EmberObject and make it easier to switch to native class syntax.

## Transition Path

As mentioned in the motivation, the `@ember/objects/events` package mostly exposes the functionality of the Evented mixin (aside from the method `Evented.has`)

There is a straightforward mapping from an Evented mixin method to a function provided by `@ember/object/events` as listed below:

* On:  
`eventedObj.on(eventName, target, method)` -> `addListener(eventedObj, eventName, target, method)`

* One:  
`eventedObj.one(eventName, target, method)` -> `addListener(eventedObj, eventName, target, method, true)`

* Off:  
`eventedObj.off(eventName, target, method)` -> `removeListener(eventedObj, eventName, target, method)`

* Trigger:  
`eventedObj.trigger(eventName, param1, param2, ..., paramN)` -> `sendEvent(eventedObj, eventName, [param1, param2, ..., paramN])`

* Has:  
`eventedObj.has(eventName)` -> `hasListeners(eventedObj, eventName)`  
Currently `hasListeners` is not a function that can be imported from `@ember/object/events`. `hasListeners` would need to be exported for this deprecation to be possible.

To assist with this transition, it should be possible to provide a codemod that updates the application to make use of the events functions.

## How We Teach This

The guides currently make no mention of the Evented mixin. Thus, the best place to teach this would be in the API documentation for the Evented mixin.

The Evented class could be marked as deprecated, and the class documentation could provide direction to use the `@ember/object/events` package instead.

Each function in the Evented Mixin (`has`, `off`, `on`, `one`, `trigger`) could be deprecated. An example of how to modify the Evented method to make use of the corresponding functions exposed in the `@ember/object/events` package would be great.

For example (from the documetation of the `trigger` method):

```
person.on('didEat', function(food) {
    console.log('person ate some ' + food);
});

person.trigger('didEat', 'broccoli');

// outputs: person ate some broccoli
```

could be written as

```
import { addListener, sendEvent } from '@ember/object/events';

...

addListener(person, 'didEat', function(foo) {
    console.log('person ate some ' + food);
});

sendEvent(person, 'didEat', ['brocolli']);

// outputs: person ate some broccoli
```


It should be possible to provide a codemod, to assist with this transition.

## Drawbacks

This RFC introduces churn in existing apps that make use of the Evented Mixin.

## Alternatives

Do not deprecate the Evented Mixin

## Unresolved questions

None
