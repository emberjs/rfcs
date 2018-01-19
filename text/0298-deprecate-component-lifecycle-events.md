- Start Date: 2018-01-19
- RFC PR: 0298
- Ember Issue: (leave this empty)

# Deprecate Component Lifecycle Events

## Summary

The goal of this RFC is to remove component lifecycle events, such as `Ember.on('didRender'`,
in favor of enforcing usage of the lifecycle methods themselves i.e. `didRender(){}`

## Motivation

When using events, timing is not guaranteed, so it is possible to have unintended
consequences and have things execute in an unexpected order. Using the methods directly
does guarantee timing and solidifies the path forward on one recommended way of using
lifecycle hooks.

## Detailed design

Ember will start logging deprecation messages that state which event you used and which
method it should be converted to.

The exact deprecation message will be decided later, but something along the lines of:

```
Using `Ember.on` event listeners for component lifecycle hooks is deprecated.
Please move your code from `myHandler: Ember.on('didRender'` into the `didRender(){}` method.
```

Ember does not appear to rely on these events internally, however there are at least a couple of mentions of 
these events in the comments, referring specifically to `on('didInsertElement')` 
[here](https://github.com/emberjs/ember.js/blob/master/packages/ember-metal/lib/run_loop.js#L140) 
and [here](https://github.com/emberjs/ember.js/blob/master/packages/ember-runtime/lib/ext/function.js#L149) 
We will want to update those, and any other comments to reflect the new way of doing things.

We will also likely want to add some new tests, that ensure that deprecation messages are logged, when someone attempts to use the events.

We would likely want to implement this in the same way the `didInitAttrs` deprecation was implemented. i.e. we should maintain an array of
component lifecycle hooks, and check if `hooks.includes(eventName)` and issue a deprecation if it does.

## How we teach this

The docs already recommend against using `Ember.on` for lifecycle hooks, but they do currently
still mention it is possible.

```
While didInsertElement() is technically an event that can be listened for using on(),
it is encouraged to override the default method itself, particularly when order of execution is important.
```

We will want to update the docs to either not mention using `on()` at all or to explicitly say *not* to use it.

Other than updating the docs, the deprecation messages should be explanatory enough that people understand the path
forward and convert all their `on()` style lifecycle hooks to the actual methods.

### Deprecation Guide

An entry to the [Deprecation Guides](https://emberjs.com/deprecations/) will be added outlining the conversion from
events to lifecycle hook methods directly.

Developers could historically use component lifecycle hooks directly or write `on()` listeners to fire when those hooks
fire. This proved to be unreliable, and tricky to debug, as the order of execution was not guaranteed. To fix this, we
are now recommending using the lifecycle hook methods directly.

Before:
```js
myHandler: on('didRender', () => {
  console.log('didRender!');
})
```

After:
```js
didRender() {
  this._super(...arguments);
  console.log('didRender!');
}
```

### Enforcing with ESLint

eslint-plugin-ember, which is also now used in the default ember-cli blueprints, enforces this rule as well with
[https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-on-calls-in-components.md](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-on-calls-in-components.md).

### Codemod

A codemod will be provided to allow automatic conversion of event listeners to use the lifecycle hook methods directly.

## Drawbacks

The only potential drawback, as far as I know, is that large code bases relying on lifecycle events will
experience a potentially large amount of churn as they convert things over. We could potentially provide a
codemod, if we think this will be problematic.

## Alternatives

We could continue to support events and methods and just continue to enforce not using `on()` via ESLint.

## Unresolved questions

* The exact deprecation message we want to issue when people use the deprecated events is uncertain.
* Should we provide an addon to enable legacy support of events?
