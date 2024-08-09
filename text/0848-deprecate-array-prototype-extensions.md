---
stage: released
start-date: 2022-08-21T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/848'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1020'
  released: 'https://github.com/emberjs/rfcs/pull/1042'
project-link:
---

# Deprecate array prototype extensions

## Summary

This RFC proposes to deprecate array prototype extensions.

## Motivation

Ember historically extended the prototypes of native Javascript arrays to implement `Ember.Enumerable`, `Ember.MutableEnumerable`, `Ember.MutableArray`, `Ember.Array`. This added convenient methods and properties, and also made Ember arrays automatically participate in the Ember Classic reactivity system.

Those convenient methods increase the likelihood of becoming potential roadblocks for future built-in language extensions, and make it confusing for users to onboard: is it specifically part of Ember, or Javascript? Also with Ember Octane, the new reactivity system, those classic observable-based methods are no longer needed.

We had deprecated [Functions](https://github.com/emberjs/rfcs/blob/master/text/0272-deprecation-native-function-prototype-extensions.md) and [Strings](https://github.com/emberjs/rfcs/blob/master/text/0236-deprecation-ember-string.md) prototype extensions. Array is the last step. And internally we had already been preferring generic array methods over prototype extensions ([epic](https://github.com/emberjs/ember.js/issues/15501)).

Continuing in that direction, we should consider recommending the usage of native array functions as opposed to convenient prototype extension methods, and the usage of `@tracked` properties or `TrackedArray` over classic reactivity methods.

## Transition Path

For convenient methods like `filterBy`, `compact`, `sortBy` etc., the replacement functionalities already exist either through native array methods or utility libraries like [lodash](https://lodash.com), [Ramda](https://ramdajs.com), etc.

For mutation methods (like `pushObject`, `removeObject`) or observable properties (like `firstObject`, `lastObject`) participating in the Ember classic reactivity system, the replacement functionalities also already exist in the form of immutable update style with tracked properties like `@tracked someArray = []`, or through utilizing `TrackedArray` from `tracked-built-ins`.

We don't need to build anything new specifically, however, the bulk of the transition will be
focused on deprecating the existing usages of array prototype extensions.

## How We Teach This

We should turn off `EmberENV.EXTEND_PROTOTYPES` by default for new applications.

For existing apps, a deprecation message will be displayed when `EmberENV.EXTEND_PROTOTYPES` flag is not set to `false`. Clear instructions will be provided about turning off the flag and fixing any existing breaks.

An entry to the [Deprecation Guides](https://deprecations.emberjs.com/v4.x) will be added outlining the different recommended transition strategies. ([Proposed deprecation guide](https://github.com/ember-learn/deprecation-app/pull/1192))

Rule `ember/no-array-prototype-extensions` is available for both [eslint](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-array-prototype-extensions.md) and [template lint](https://github.com/ember-template-lint/ember-template-lint/blob/master/docs/rule/no-array-prototype-extensions.md) usages. Rule examples have recommendations for equivalences.

We can leverage the fixers of lint rule to auto fix some of the issues, e.g. the built-in [fixer](https://github.com/ember-template-lint/ember-template-lint/blob/master/docs/rule/no-array-prototype-extensions.md) of `firstObject` usages in template. 

We also should create codemods or autofixers in lint rules for some of the convinient functions like `reject`, `compact`, `any` etc. More discussions on **Unresolved Questions** section.

Examples (taken from [Deprecation Guide](https://github.com/smilland/rfcs/pull/1)): 
### Convenient methods
Examples of deprecated and current code:
```js
import Component from '@glimmer/component';
import { uniqBy, sortBy } from 'lodash';

export default class SampleComponent extends Component {
  abc = ['x', 'y', 'z', undefined, 'x'];
  
  // deprecated
  def = this.abc.compact();
  ghi = this.abc.uniq();
  jkl = this.abc.toArray();
  mno = this.abc.uniqBy('y');
  pqr = this.abc.sortBy('z');
  // ...

  // current
  // compact
  def = this.abc.filter(el => el !== null && el !== undefined);
  // uniq
  ghi = [...new Set(this.abc)];
  // toArray
  jkl = [...this.abc];
  // uniqBy
  mno = uniqBy(this.abc, 'y');
  // sortBy
  pqr = sortBy(this.abc, 'z');
};
```

### Observable properties and methods in js
Examples of deprecated code:
```js
import Component from '@glimmer/component';
import { action } from '@ember/object';

export default class SampleComponent extends Component{
  abc = [];

  // observable property
  get lastObj() {
    return this.abc.lastObject;
  }

  @action
  someAction(newItem) {
    // observable method
    this.abc.pushObject(newItem);
  }
};
```

Examples of current code. 
#### Option 1: use `tracked` property
```js
import Component from '@glimmer/component';
import { action } from '@ember/object';
import { tracked } from '@glimmer/tracking';

export default class SampleComponent extends Component{
  @tracked abc = [];

  get lastObj() {
    return this.abc.at(-1);
  }

  @action
  someAction(newItem) {
    this.abc = [...abc, newItem];
  }
};
```

#### Option 2: use `TrackedArray`
```js
import Component from '@glimmer/component';
import { action } from '@ember/object';
import { TrackedArray } from 'tracked-built-ins';

export default class SampleComponent extends Component{
  abc = new TrackedArray();

  get lastObj() {
    return this.abc.at(-1);
  }

  @action
  someAction(newItem) {
    abc.push(newItem);
  }
};
```

### Observable properties in templates
Examples of deprecated code:

```hbs
<Foo @bar={{@list.firstObject.name}} />
```

Examples of current code:
```hbs
<Foo @bar={{get @list '0.name'}} />
```

After the deprecated code is removed from Ember (at 5.0), we need to remove the [options to disable the array prototype extension](https://guides.emberjs.com/v4.2.0/configuring-ember/disabling-prototype-extensions/) from Official Guides and we also need to update the [Tracked Properties](https://guides.emberjs.com/v4.2.0/upgrading/current-edition/tracked-properties/#toc_arrays) `Arrays` section with updated suggestions.

## Drawbacks
- For users relying on Ember array prototype extensions, they will have to refactor their code and use equivalences appropriately.
- A lot of existing Ember forums/blogs had been assuming the enabling of array prototype extensions which could cause confusions for users referencing them.
- Increase package sizes, for example, before `this.abc.filterBy('x');`, now `this.abc.filter(el => el !== 'x');`.
- Although `tracked-built-ins` is on the path to stabilization as an official API via [RFC #812](https://github.com/emberjs/rfcs/pull/812), it is not yet officially recommended and its API may change somewhat between now and stabilization.

## Alternatives
- Do the deprecation as suggested here for use within Ember itself, but extract it as a standalone library for users who want to use it. This will only work as long as the underlying Ember Classic reactivity system is supported.

    As a variation on this, we could do this but explicitly only support it up through the first LTS release in the 5.x series.

- Continuing allowing array prototype extensions but turning the `EXTEND_PROTOTYPES` off by default.

- Do one of these, but target Ember v6 instead.

- Do nothing.

## Unresolved questions
- The current lint rule [ember/no-array-prototype-extensions](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-array-prototype-extensions.md) has some false positives because static analysis can't always know when a given object is a native array vs. an `EmberArray` (which has the same array convenience methods) vs. another class with overlapping method names (although the rule does employ some heuristics to avoid false positives when possible).
- Difficulties for providing stable codemods or autofixers:
  1. giving false positives on lint rules, same will apply to codemods or autofixers;
  2. when migrating certain methods, we need to access object. Shall we use Ember `get` or native way? Unless we fully remove ObjectProxy dependency, codemods or autofixers would still require manual work in certain cases.
  3. observable functions or properties requires manual refactoring;
