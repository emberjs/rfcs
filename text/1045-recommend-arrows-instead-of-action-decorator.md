---
stage: accepted
start-date: 2024-09-23T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1045 # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

<-- Replace "RFC title" with the title of your RFC -->
# Recommend arrow function in classes instead of using the `@action` decorator 

## Summary

The `@action` decorator has been a pivotal tool for helping older ember projects migrate to native classes. 
However, needing to import something _just_ to have a bound `this` in a method on a class feels unergonomic.

This RFC proposes we no longer recommend `@action` in the documentation, and instead use arrow functions assigned to a class property.

## Motivation

When prototyping, or "writing code quickly", it's much easier to type out an arrow function than it is to import something to apply to your class method.
We want to support fast and correct authoring of components, so that means we need to reduce "unneeded" imports -- one such import is the `@action` decorator.

Arrow functions have been in JavaScript for ~ 10 years now (since 2015) _and_ the amount of legacy code relying on the compatibility behavior supported by `@action` is _significantly_ less than it was.

For example a class with a bound "function":

```js
class Demo {
    msg = 'hi';
    greet = () => console.log(this.msg);
}
```
the "function" is a property where the value of that property _happens_ to be a function, relying on fact that, on the right-hand side of an equals inherits the instance-`this` of the class.

As were typing out the same thing with the current recommendations, we run in to a bit of mental "flow state" interruption:
```js 
// 2: type the import
import { action } from '@ember/object';

class Demo {
    msg = 'hi'

    // 3: type the action decorator
    @action
    greet() {
        // 1: INTERRUPT: have to import @action
        console.log(this.msg);
    }
```
This is not ideal.


Additionally, a move to arrow functions would remove one more "emberism" and align with _The Platform_, reducing the amount of things folks would be expected to learn / know about as they learn Ember.


## Detailed design

In [guides-source](https://github.com/ember-learn/guides-source), there are 112 occurances of `@action`.

All of these should be replaced by arrow functions. But first, what is the trade-off we're making?

With traditional methods on a JavaScript class, e.g.:
```js 
class Demo {
    greet() {}
}
```
the `greet` method is on `Demo`'s prototype, and is shared between all instances. This is great for memory efficiency.
However, when passing the `greet` method somewhere to be called later [such as the glimmer-vm][^vm-fix], the `this` that is used is that of the surrounding context, not the originating class-instance.

[^vm-fix]: Note that this specific case will be fixed in the GlimmerVM as we can make the Glimmer VM interpret `this.method` as `method.call(this, ...args)`, which eliminates the need to bind `this` in classes that directly invoke a method. This would _not_ change any calling behavior when _passing the function_, e.g.: `<Form @onSubmit={{this.submit}} />`. When passing a function like this, it _must_ be bound if `submit` accesses any `this`. We _could_ also swap this out at compile time to `this.submit.bind(this)` -- but this is a separate change and a separate evaluation of risks is needed.

The [`@action`](https://github.com/emberjs/ember.js/blob/v5.11.0/packages/%40ember/object/index.ts#L224) decorator solves this by, at class-evaluation-time, swapping the definition of the defined method, with a [getter](https://github.com/emberjs/ember.js/blob/v5.11.0/packages/%40ember/object/index.ts#L184) that, per-instance, maintains a [`WeakMap` of bound / duplicate-per-instance functions](https://github.com/emberjs/ember.js/blob/v5.11.0/packages/%40ember/object/index.ts#L205-L210) incurs [additional runtime cost](https://github.com/emberjs/ember.js/blob/v5.11.0/packages/%40ember/object/index.ts#L212-L219) for each call to a function.


`@action` is only _needed_ when we know that the `this` context would be lost. Historically this has been taught as "when you invoke a function from a template, or pass a function somewhere else".
We can use that same heuristic for "when to define an arrow function", and **this should be mentioned in the guides**, along with a sample error message "can't access x on undefined" (though, I think some browsers would also print `this.x` in their error message, giving more context / hints to the fact that `this` is undefined. More sinisterily, though, the `this` could be the _wrong_ `this`. This is a hazard of working with `this` in JavaScript, and why some people liberally used `@action` on functions, so they didn't have to worry about this problem at all. Because of liberal usage of `@action`, the above-described runtime cost grows more than it needs to. Do note that this cost is small.

With arrow functions, we completely eliminate all of the memory upkeep that the `@action` decorator is doing. We still duplicate the methods per-instance (those that are defined as arrows), but all the upkeep around those methods is _handled for us_ by the JavaScript engine. In modern code, using arrow functions is kind of the equivelent of only having [this line](https://github.com/emberjs/ember.js/blob/v5.11.0/packages/%40ember/object/index.ts#L215C13-L215C14) from the `@action` decorator.

## How we teach this

1. Update all usages of `@action` to be arrows in the guides, removing all imports of the `@action` decorator.
2. On the ["Component State and Actions"](https://guides.emberjs.com/release/components/component-state-and-actions/) page (where action is first used), explain the purpose of the arrow function, and why we use it. (this explanation is currently missing for `@action` as well -- however at the bottom of the page, it links to ["Patterns for Actions"](https://guides.emberjs.com/release/in-depth-topics/patterns-for-actions/) -- which _also_ does not mention anything about this-binding. We can link out to MDN ([maybe this one](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this#bound_methods_in_classes) -- or we could write something ourselves for a stable-reference to that content).  We should also explain the advantages of _not_ using an arrow function (shared memory between instances).

## Drawbacks

No compatibility with legacy code (pre Octane)

## Alternatives

Do nothing.

## Unresolved questions

None.
