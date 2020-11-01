---
Start Date: 2017-11-20
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/272
Tracking: https://github.com/emberjs/rfc-tracking/issues/12

---

# Deprecate Function.prototype.on, Function.prototype.observes and Function.prototype.property

## Summary

This RFC proposes to deprecate `Function.prototype.on`,
`Function.prototype.observes` and `Function.prototype.property`

## Motivation

Ember has been moving away from extending native prototypes due to the confusion
that this causes users: is it specifically part of Ember, or JavaScript?

Continuing in that direction, we should consider recommending the usage of
[`on` (`@ember/object/evented`)](https://emberjs.com/api/ember/2.18/classes/@ember%2Fobject%2Fevented/methods/on?anchor=on), [`observer` (`@ember/object`)](https://emberjs.com/api/ember/2.18/classes/@ember%2Fobject/methods/observer?anchor=observer) and [`computed` (`@ember/object`)](https://emberjs.com/api/ember/2.18/classes/@ember%2Fobject/methods/computed?anchor=computed) as opposed to their native
prototype extension equivalents.
We go from two ways to do something, to one.

[`eslint-plugin-ember` already provides this as a rule](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-function-prototype-extensions.md).

## Transition Path

The replacement functionality already exists in the form of `on`, `observer`, and `computed`.

We don't need to build anything new specifically, however, the bulk of the transition will be
focused on deprecating the native prototype extensions.

A codemod for this deprecation has to take into consideration that while `foo: function() { /* custom logic */ }.property('bar')` is a `Function.prototype` extension, `foo: observer(function () { /* some custom logic */ }).on('customEvent')` is not.

## How We Teach This

On the deprecation guide, we can showcase the same example as above. We can explain why
the proposal was necessary, followed by a set of examples highlighting the deprecated
vs current style.

Borrowing from the [ESLint plugin example](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-function-prototype-extensions.md):

```js
import { computed, observer } from '@ember/object';
import { on } from '@ember/object/evented';

export default Component.extend({
  // deprecated
  abc: function() { /* custom logic */ }.property('xyz'),
  def: function() { /* custom logic */ }.observe('xyz'),
  ghi: function() { /* custom logic */ }.on('didInsertElement'),
  jkl: function() { /* custom logic */ }.on('customEvent'),

  // current
  abc: computed('xyz', function() { /* custom logic */ }),
  def: observer('xyz', function() { /* custom logic */ }),
  didInsertElement() { /* custom logic */ }),
  jkl: on('customEvent', function() { /* custom logic */ }),
});
```

The official Guides currently [discourage the use of `Function.prototype` extensions](https://guides.emberjs.com/v2.17.0/configuring-ember/disabling-prototype-extensions/):

> Function is extended with methods to annotate functions as computed properties,
> via the property() method, and as observers, via the observes() method. Use of
> these methods is now discouraged and not covered in recent versions of the Guides.

After the deprecated code is removed from Ember, we need to remove the section
about `Function` prototypes altogether.

## Alternatives

None.
