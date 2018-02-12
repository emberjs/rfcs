- Start Date: 2017-11-20
- RFC PR: https://github.com/emberjs/rfcs/pull/272
- Ember Issue:

# Summary

This RFC proposes to deprecate `Function.prototype.on`,
`Function.proptotype.observes` and `Function.prototype.property`

# Motivation

Ember has been moving away from extending native prototypes due to the confusion
that this causes users: is it specifically part of Ember, or JavaScript?

Continuing in that direction, we should consider recommending the usage of
[`on` (`@ember/object/evented`)](https://emberjs.com/api/ember/2.18/classes/@ember%2Fobject%2Fevented/methods/on?anchor=on), [`observer` (`@ember/object`)](https://emberjs.com/api/ember/2.18/classes/@ember%2Fobject/methods/observer?anchor=observer) and [`computed` (`@ember/object`)](https://emberjs.com/api/ember/2.18/classes/@ember%2Fobject/methods/computed?anchor=computed) as opposed to their native
prototype extension equivalents.
We go from two ways to do something, to one.

[`eslint-plugin-ember` already provides this as a rule](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-function-prototype-extensions.md).

# Transition Path

The replacement functionality already exists in the form of `on`, `observer`, and `computed`.
We don't need to build anything new specifically, however, the bulk of the transition will be
focused on deprecating the native prototype extensions.

Borrowing from the [ESLint plugin example](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-function-prototype-extensions.md):

```js
import { computed, observer } from '@ember/object';
import { on } from '@ember/object/evented';

export default Component.extend({
  // current
  abc: function() { /* custom logic */ }.property('xyz'),
  def: function() { /* custom logic */ }.observe('xyz'),
  ghi: function() { /* custom logic */ }.on('didInsertElement'),

  // proposed
  abc: computed('xyz', function() { /* custom logic */ }),
  def: observer('xyz', function() { /* custom logic */ }),
  ghi: on('didInsertElement', function() { /* custom logic */ }),
});
```

We need to create a codemod that will transform code from the `current` form to the
`proposed` form.

# How We Teach This

On the deprecation guide, we showcase the same example as above. We can explain why
the proposal was necessary, followed by deprecated to current code examples.

The Guides currently discourage the use of `Function` prototype extensions.
For example, from the [disabling prototype extensions page](https://guides.emberjs.com/v2.17.0/configuring-ember/disabling-prototype-extensions/):

After the deprecated code is removed from Ember, the same page needs to remove the section
about `Function` prototypes altogether.

# Alternatives

None.
