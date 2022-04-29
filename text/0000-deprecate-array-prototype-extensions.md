---
Stage: Initial
Start Date: 2022-4-01
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: 
---

# Deprecate array prototype extensions

## Summary

This RFC proposes to deprecate array prototype extensions.

## Motivation

Ember historically extended the prototypes of native Javascript arrays to implement `Ember.Enumerable`, `Ember.MutableEnumerable`, `Ember.MutableArray`, `Ember.Array`. This added convenient methods and properties, and also made Ember arrays automatically participate in the Ember Classic reactivity system.

Those convenient methods increase the likelihood of becoming potential roadblocks for future built-in language extensions, and make it confusing for users to onboard: is it specifically part of Ember, or Javascript? Also with Ember Octane, the new reactivity system, those classic observable-based methods are no longer needed.

We had deprecated [Functions](https://github.com/emberjs/rfcs/blob/master/text/0272-deprecation-native-function-prototype-extensions.md) and [Strings](https://github.com/emberjs/rfcs/blob/master/text/0236-deprecation-ember-string.md) prototype extensions. Array is the last step. And internally we had already been preferring generic array methods over prototype extensions ([epic](https://github.com/emberjs/ember.js/issues/15501)).

Continuing in that direction, we should consider recommending the usage of native array functions as opposed to convenient prototype extension functions, and the usage of new tracked properties or `TrackedArray` over classic reactivity methods.

## Transition Path

For convenient methods like `without`, `sortBy`, `uniqBy` etc., the replacement functionality already exists either through generic array functions or utility libraries like [lodash](https://lodash.com), [Ramda](https://ramdajs.com), etc.

For helper functions participating in the Ember classic reactivity system like `pushObject`, `removeObject`, the replacement functionality also already exists in the form of immutable update style with tracked propreties like `@tracked someArray = []`, or through utilizing `TrackedArray` from `tracked-built-ins`.

During transition, we still allow users to use `A` from `@ember/array`.

We don't need to build anything new specifically, however, the bulk of the transition will be
focused on deprecating the array prototype extensions.

## How We Teach This

An entry to the [Deprecation Guides](https://deprecations.emberjs.com/v4.x) will be added outlining the different recommended transition strategies.

Rule `ember/no-array-prototype-extensions` is available for both [eslint](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-array-prototype-extensions.md) and [template lint](https://github.com/ember-template-lint/ember-template-lint/blob/master/docs/rule/no-array-prototype-extensions.md). Rule examples have recommendations for equivalences.

We can leverage the fixers of lint rule to auto fix some of the issues. We will also create codemods to help with this migration.

### Convenient methods
Examples of deprecated and current code:
```js
import Component from '@glimmer/component';
import { uniqBy, sortBy } from 'lodash';

export default class SampleComponent extends Component{
  abc = ['x', 'y', 'z', 'x'];
  
  // deprecated
  def = this.abc.without('x');
  ghi = this.abc.uniq();
  jkl = this.abc.toArray();
  mno = this.abc.uniqBy('y');
  pqr = this.abc.sortBy('z');
  // ...

  // current
  def = this.abc.filter(el => el !== 'x');
  ghi = [...new Set(this.abc)];
  jkl = [...this.abc];
  mno = uniqBy(this.abc, 'y');
  pqr = sortBy(this.abc, 'z');
};
```

### Observable based methods
Examples of deprecated code:
```js
import Component from '@glimmer/component';
import { action } from '@ember/object';

export default class SampleComponent extends Component{
  abc = [];

  @action
  someAction(newItem) {
    this.abc.pushObject('1');
  }
};
```

Examples of current code. 
#### Option 1: tracked properties
```js
import Component from '@glimmer/component';
import { action } from '@ember/object';
import { tracked } from '@glimmer/tracking';

export default class SampleComponent extends Component{
  @tracked abc = [];

  @action
  someAction(newItem) {
    this.abc = [...abc, newItem];
  }
};
```

#### Option 2: `TrackedArray`
```js
import Component from '@glimmer/component';
import { action } from '@ember/object';
import { tracked } from '@glimmer/tracking';
import { TrackedArray } from 'tracked-built-ins';

export default class SampleComponent extends Component{
  @tracked abc = new TrackedArray();

  @action
  someAction(newItem) {
    abc.push(newItem);
  }
};
```

### Properties in templates
Examples of deprecated code:

```hbs
<Foo @bar={{@list.firstObject.name}} />
```

Examples of  current code.
```hbs
<Foo @bar={{get @list '0.name'}} />
```

After the deprecated code is removed from Ember (at 5.0), we need to remove the [options to disable the array prototype extension](https://guides.emberjs.com/v4.2.0/configuring-ember/disabling-prototype-extensions/) from Official Guides and we also need to update the [Tracked Properties](https://guides.emberjs.com/v4.2.0/upgrading/current-edition/tracked-properties/#toc_arrays) `Arrays` section with updated suggestions.

## Drawbacks
- For users relying on Ember array prototype extensions, they will have to refactor their code and use equivalences appropriately.
- A lot of existing Ember forums/blogs had been assuming the enabling of array prototype extensions which could cause confusions for users referencing them.
- Increase package sizes, for example, before `this.abc.filterBy('x');`, now `this.abc.filter(el => el !== 'x');`.

## Alternatives
- Continuing allowing array prototype extensions but turning the EXTEND_PROTOTYPES off by default.
- Do nothing.

### Unresolved questions
- Some methods like `removeObject`, `removeObjects` that have non-trival logics might require custom helper function in application to ease the migration.
- `replace` method is one of ember array extensions but also be string native proptypes, which adds uncertainty for lint rules and can cause issues during the deprecation.
- Shall we deprecate `Ember.A()` as well or still allowing users to use it?
