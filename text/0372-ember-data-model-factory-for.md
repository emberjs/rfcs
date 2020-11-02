---
Start Date: 2018-09-06
Relevant Team(s): Ember Data
RFC PR: https://github.com/emberjs/rfcs/pull/372
Tracking: https://github.com/emberjs/rfc-tracking/issues/16

---

# ember-data | modelFactoryFor

## Summary

Promote the private `store._modelFactoryFor` to public API as `store.modelFactoryFor`.

## Motivation

This RFC is a follow-up RFC for [#293 RecordData](https://github.com/emberjs/rfcs/pull/293).

Ember differentiates between `klass` and `factory` for classes registered with the container.
 At times, `ember-data` needs the `klass`, at other times, it needs the `factory`. For this reason,
`ember-data` has carried two APIs for accessing one or the other for some time. The public `modelFor`
 provides access to the `klass` where schema information is stored, while the private `_modelFactoryFor`
 provides access to the factory for instantiation.
 
We provide access to the class with `modelFor` roughly implemented as `store._modelFactoryFor(modelName).klass`.
We instantiate records from this class roughly implemented as `store._modelFactoryFor(modelName).create({ ...args })`.

For symmetry, both of these APIs should be public. Making `modelFactoryFor` public would provide a hook
 that consumers can override should they desire to provide a custom `ModelClass` as an alternative
 to `DS.Model`.

## Detailed design
Due to previous complexity in the lookup of models in `ember-data`, we previously had both `modelFactoryFor`
and `_modelFactoryFor`. Despite the naming, both of these methods were private. During a recent cleanup phase,
we unified the methods into `_modelFactoryFor` and left a deprecation in `modelFactoryFor`. This RFC proposes
un-deprecating the `modelFactoryFor` method and making it public, while deprecating the private `_modelFactoryFor`.

More precisely:

- `store._modelFactoryFor` becomes deprecated and calls `store.modelFactoryFor`.
- `store.modelFactoryFor` becomes un-deprecated.

### The contract for `modelFactoryFor`

The return value of `modelFactoryFor` MUST be the result of a call to [`applicationInstance.factoryFor`](https://www.emberjs.com/api/ember/3.4/classes/ApplicationInstance/methods/factoryFor?anchor=factoryFor)
 where `applicationInstance` is the `owner` returned by using `getOwner(this)` to access the `owner` of the `store` instance.

```typescript
interface Klass {}

interface Factory {
  klass: Klass,
  create(): Klass
}

interface FactoryMap {
    [factoryName: string]: Factory
}

declare function factoryFor<K extends keyof FactoryMap>(factoryName: K): FactoryMap[K];

interface Store {
  modelFactoryFor(modelName: string): ReturnType<typeof factoryFor>;
}
```

Users interested in providing a custom class for their `records` and who override `modelFactoryFor`,
 would not need to also change `modelFor`, as this would be the `klass` accessible via the `factory`.

Users wishing to extend the behavior of `modelFactoryFor` could do so in the following manner:

**Example 1:**

**services/store.js**
```js
import { getOwner } from '@ember/application';
import Store from 'ember-data/store';

export default Store.extend({
  modelFactoryFor(modelName) {
    if (someCustomCondition) {
      return getOwner(this).factoryFor(someFactoryName);
    }
    
    return this._super(modelName);
  }
});
```
 
 #### `Model.modelName`
 
 `ember-data` currently sets `modelName` onto the `klass` accessible via the `factory`. For classes that do not
   inherit from `DS.Model` this would not be done, although end users may do so themselves in their implementations
   if so desired.

### What is a valid factory?

The default export of a custom ModelClass **MUST** conform to the requirements of `Ember.factoryFor`. The requirements
 of `factoryFor` are currently underspecified; however, in practice, this means that the default export is an
 instantiable class with a static `create` method and an instance `destroy` method or that inherits from `EmberObject`
 (which provides such methods).
 
**Example 2:**

```javascript
import { assign } from '@ember/polyfills';

export default class CustomModel {
  constructor(createArgs) {
    assign(this, createArgs);
  }
  destroy() {
    // ... do teardown
  }
  static create(createArgs) {
    return new this(createArgs);
  }
}
```

**Example 3:**

```javascript
import EmberObject from '@ember/object';

export default class CustomModel extends EmberObject {
  constructor(createArgs) {
    super(createArgs);
  }
}
```

Custom classes for models should expect their constructor to receive a single argument: an object with *at least*
 the following.
 
- A `recordData` instance accessible via `getRecordData` (see below)
- Any properties passed as the second arg to `createRecord`
- An `owner` accessible via `Ember.getOwner`
- Any DI injections
- any other properties that `Ember` chooses to pass to a class instantiated via `factory.create` (currently none)

### getRecordData

Every `record` (instance of the class returned by `modelFactoryFor`) will have an associated [RecordData](https://github.com/emberjs/rfcs/pull/293)
 which contains the backing data for the id, type, attributes and relationships of that record.
 
This backing data can be accessed by using the `getRecordData` util on the `record` (or on the `createArgs` passed to
 a record). Using `getRecordData` on a `record` is only guaranteed after the record has been instantiated. During
 instantiation, this call should be made on the `createArgs` object passed into the record.
 
**Example 4**

```javascript
import { getRecordData } from 'ember-data';

export default class CustomModel {
  constructor(createArgs) {
    // during instantiation, `recordData` is available by calling `getRecordData` on createArgs
    let recordData = getRecordData(createArgs);
  }
  someMethod() {
    // post instantiation, `recordData` is available by calling `getRecordData` on the instance
    let recordData = getRecordData(this);
  }
  destroy() {
    // ... do teardown
  }
  static create(createArgs) {
    return new this(createArgs);
  }
}
```

## How we teach this

This API would be intended for addon-authors and power users. It is not expected
that most apps would implement custom models, much as it is not expected that most
apps would implement custom `RecordData`. The teaching story would be limited to
documenting the nature and purpose of `modelFactoryFor`.

## Drawbacks

- Users may try to use the hook to instantiate records on their own. Ultimately, the store
  should still do the instantiating.

## Alternatives

Users could define models in `models/*.js` that utilize a custom `ModelClass`.
However, such an API for custom classes would exclude the ability to dynamically
generate classes.

## Unresolved questions

None
