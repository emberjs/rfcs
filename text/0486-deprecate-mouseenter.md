- Start Date: 2019-04-28
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/486
- Tracking: 

# Deprecate support for mouseEnter/Leave/Move Ember events

## Summary

Deprecate support for `mouseenter`, `mouseleave` and `mousemove` events in Ember's EventDispatcher. This affects
the corresponding event handler methods (`mouseEnter() {}`) in Ember components and 
`{{action "some" on="mouseenter"}}`. 

## Motivation

Ember's EventDispatcher handles "Ember events" by attaching listeners to the app's root element
and relying on the events bubbling up to that element (aka event delegation). There they 
are processed and invoke any matching `Ember.Component` event handler method with 
the same (camel-cased) method as the event type. Same for element-space `{{action}}` 
modifiers.

> Note: for a "Deep Dive on Ember Events" and how they differ from "native events" I refer to 
Marie Chatfield's excellent 
[blog post](https://medium.com/square-corner-blog/deep-dive-on-ember-events-cf684fd3b808)
or [EmberConf talk](https://youtu.be/G9hXjjHFJVs)

This works fine in general, but `mouseenter`/`mouseleave` events are special as they do
not bubble. In the past it still worked nevertheless as jQuery transparently handled this
for us, as it had special support for event delegation for these events, essentially by using 
(bubbling) `mouseover` events to replicate the semantics of `mouseenter`/`mouseleave` events.

When support for jQuery-less apps was introduced, this [left a hole](https://github.com/emberjs/ember.js/issues/16591)
in the jQuery-less EventDispatcher implementation. But as support for those events was and
still is part of Ember's pubic API, we had no chance other than to [implement support
for jQuery-less apps](https://github.com/emberjs/ember.js/pull/16603) using the same 
`mouseover` based approach.

This however comes with a cost: besides an [unresolved issue](https://github.com/emberjs/ember.js/issues/17228)
the implementation has some performance drawbacks, as it has to process every `mouseover` event on 
*any* element, create fake `mouseenter`/`mouseleave` events and try to dispatch them, even when 
not a single component/action needs them.  

Deprecating support for `mousemove` is also proposed, which is a (bubbling) event that does not have the higher 
implementation cost as `mouseenter`/`mouseleave`, but nevertheless requires the EventDispatcher to optimistically handle
these extremely high-volume events.

While efforts to make this more "pay as you go" are [possible](https://github.com/emberjs/ember.js/pull/17911), 
the trade-off of keeping support around still seems unfavorable, as these events fire so
frequently, while they are (most certainly) very rarely used.

This is even more so given that Glimmer Components with their outerHTML semantics do not 
work with event handler methods, and `{{action}}` will eventually fade away in favor of
`{{on}}` using native `addListener()`.

## Transition Path

When a `mouseenter`/`mouseleave`/`mousemove` event is dispatched by the EventDispatcher to a component
or `{{action}}` listening to that event type, a deprecation warning will be issued.

The linked migration guide will cover these use cases:

#### Ember.Component

Before:

```js
import Component from '@ember/component';

export default Component.extend({
  mouseEnter(e) {
    // do something
  }
});
```

After:

```js
import Component from '@ember/component';
import { bind } from '@ember/runloop';

export default Component.extend({
  mouseenter(e) {
    // do something
  },
  didInsertElement() {
    this._super(...arguments);
    this._mouseenter = bind(this, this.mouseenter);
    this.element.addEventListener('mouseenter', this._mouseenter);
  },
  willDestroyElement() {
    this._super(...arguments);
    this.element.removeEventListener('mouseenter', this._mouseenter);
  }
});
```

#### `{{action}}` modifier

Before:

```hbs
<button {{action "handleMouseEnter" on="mouseenter"}}>Hover</button>
```

After (based on [RFC471](https://github.com/emberjs/rfcs/blob/master/text/0471-on-modifier.md)):

```hbs
<button {{on "mouseenter" this.handleMouseEnter}}>Hover</button>
```

## How We Teach This

The deprecation guide should explain the transition path as shown above.

The Guide's [Handling Events](https://guides.emberjs.com/release/components/handling-events/#toc_event-names)
section must remove the reference to `mouseenter`, `mouseleave` and `mousemove.

Other than that, no changes are required, as the replacement APIs are all available and
well established, and this RFC does not introduce anything new. Also having certain event
types not be supported by the EventDispatcher is not something new, as other high-volume
events like `mouseover` or `scroll` are also not supported (by default).

## Drawbacks

It requires refactoring existing code that relies on support for these events. But given that
these events are (presumably) rarely used, and the alternatives are easy to implement, this
should not be a major issue.

## Alternatives

We could keep support in place, and eventually work on optimizations that minimize the 
performance impact.

## Unresolved questions

None at this point.
