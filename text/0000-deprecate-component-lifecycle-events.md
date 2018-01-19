- Start Date: (fill me in with today's date, 2018-01-19)
- RFC PR: (leave this empty)
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

eslint-plugin-ember, which is also now used in the default ember-cli blueprints, enforces this rule as well with
[https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-on-calls-in-components.md](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-on-calls-in-components.md).

## Drawbacks

The only potential drawback, as far as I know, is that large code bases relying on lifecycle events will
experience a potentially large amount of churn as they convert things over. We could potentially provide a
codemod, if we think this will be problematic.

## Alternatives

This decision seems to be fairly universally supported, and I am not aware of any proposed alternatives.

## Unresolved questions

The only potential uncertainty is the exact deprecation message we want to issue when people use the
deprecated events.
