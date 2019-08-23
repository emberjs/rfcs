- Start Date: 2019-08-15
- Relevant Team(s): Ember.js, Learning
- RFC PR: https://github.com/emberjs/rfcs/pull/528
- Tracking: (leave this empty)

# Deprecate Eventing from Ember

## Summary

Deprecation support for Events within Ember.

This includes deprcating the `Evented` Mixin, and methods exposed in the `@ember/object/events` and `@ember/object/evented` packages

## Motivation

Eventing is something that is no longer core to Ember, as eventing is no longer used internally after changes from all the work on Octane.

One pillar of the Ember 2019-2020 roadmap is to "Continue Simplifying Ember". This deprecation aligns with that pillar as deprecating eventing will reduce the surface area of Ember and ensure that Ember is not maintaining a feature that is available and stable in several other libraries.

## Transition Path

If a developer still wishes to use the eventing pattern, the recommendation is to use a third-party library.

There are several eventing libraries available which are stable and well supported. Some examples include, but are not limited to:
* events (https://www.npmjs.com/package/events)
* eventemitter3 (https://www.npmjs.com/package/eventemitter3)
* event-kit (https://www.npmjs.com/package/event-kit)

Any objects that currently make use eventing pattern will need to create an instance of an emitter and use that emitter instance to subscribe to and emit events.

For example this code which makes use of the Evented mixin:

```
import Evented, { on } from `@ember/object/evented';

const EventedObject = EmberObject.extend(Evented, {
    eventHandler: on('outputMessage', function() {
        console.log('Outputting message from handler inside object');
    })
});
const eventedObj = EventedObject.create({})

const handler = (params1, param2) => {  console.log(`Let's ${param1} ${param2}!`); };
eventedObj.on('outputMessage', handler);
eventedObj.trigger('outputMessage', 'Deprecate', 'Events');
/* console log
 * Outputting message from handler inside object
 * Let's Deprecate Events!
 */
eventedObj.off('outputMessage', handler)
```

and this functionally equivalent code which makes use of the methods from the `@ember/object/events` package:

```
import { addListener, removeListener, sendEvent } from `@ember/object/events;

const EventedObject = EmberObject.extend({
    init() {
        addListener(this, 'outputMessage', this, 'eventHandler');
    },

    eventHandler() {
        console.log('Outputting message from handler inside object');
    }
});
const eventedObj = EventedObject.create({});

const handler = () => {  console.log('Outputting message from handler outside object'); };
addListener(eventedObj, 'outputMessage', handler);
sendEvent(eventedObj, 'outputMessage', ['Deprecate', 'Events']);
/* console log
 * Outputting message from handler inside object
 * Let's Deprecate Events!
 */
removeListener(eventedObj, 'outputMessage', handler);
```

can be rewritten using the third-party library `eventemitter3` as such

```
import EventEmitter from 'eventemitter3';

const EventedObject = EmberObject.extend({
    init() {
        this.emitter = new EventEmitter();
        this.emitter.on('outputMessage', this.eventHandler, this);
    },

    eventHandler() {
        console.log('Outputting message from handler inside object');
    }
});

const eventedObj = EventedObject.create({});
const handler = (params1, param2) => {  console.log(`Let's ${param1} ${param2}!`); };
eventedObj.emitter.on('outputMessage', handler);
eventedObj.emitter.emit('outputMessage', 'Deprecate', 'Events');
/* console log
 * Outputting message from handler inside object
 * Let's Deprecate Events!
 */
eventedObj.emitter.off('outputMessage', handler);
```

To be more specific each Evented mixin method and each function exported by `@ember/object/events` has a straightforward mapping to a method in the `eventemitter3` library as follows:

* on/addListener:  
`eventedObj.on(eventName, target, method)` OR `addListener(eventedObj, eventName, target, method)` OR `handler: on(eventName, method)` corresponds to
```
eventedObj.emitter.on(eventName, method, target)
```

* one/addListener:  
`eventedObj.one(eventName, target, method)` or `addListener(eventedObj, eventName, target, method, true)` corresponds to
```
eventedObj.emitter.once(eventName, method, target);
```

* off/removeListener:  
`eventedObj.off(eventName, target, method)` or `removeListener(eventedObj, eventName, target, method)` corresponds to
```
eventedObj.emitter.off(eventName, method, target);
```

* trigger/sendEvent:
`eventedObj.trigger(eventName, param1, param2, ..., paramN)` or `sendEvent(eventedObj, eventName, [param1, param2, ..., paramN])` corresponds to
```
eventedObj.emitter.emit(eventName, param1, param2, ..., paramN);
```

* has:  
`eventedObj.has(eventName)` would correspond to
```
eventedObj.emitter.listenerCount(eventName) > 0
```

## How We Teach This

The guides currently make no mention of the Evented mixin or the methods exposed in `@ember/objects/events`. Thus, the best place to teach this would be in the API documentation.

The Evented Mixin and each function in the Evented Mixin (`has`, `off`, `on`, `one`, `trigger`) would be marked as deprecated and the mixin/function documentation could point to this RFC or explicitly include an example of how to make use of a third-party library instead.
Similarly, each function exported from `@ember/object/events` could be deprecated and include an example of how to make use of a third-party library.

For example (from the documetation of the `trigger` method):

```
const Person = EmberObject.extend(Evented);
const person = PersonCreate();

person.on('didEat', function(food) {
    console.log('person ate some ' + food);
});

person.trigger('didEat', 'broccoli');

// outputs: person ate some broccoli
```

could be written as

```
import EventEmitter from 'event';

const Person = EmberObject.extend({
    init() {
        this._super(...arguments);
        this.emitter = new EventEmitter()
    }
}

person.emitter.on('didEat', function(foo) {
    console.log('person ate some ' + food);
});

person.emitter.emit('didEat', 'brocolli');

// outputs: person ate some broccoli
```

## Drawbacks

This RFC introduces churn in existing apps that make use of the Evented Mixin and Events.

Due to transitioning away from an Ember package and using a third-party library of the developers choice, this seems like it will be difficult / impossible to codemod and be a very manual process. 

## Alternatives

* Do not deprecate Ember Events
* Deprecate only the `@ember/object/evented` package and Evented Mixin, and keep the `@ember/object/events` package

## Unresolved questions

Can this process be somwhat automated by providing a codemod?
