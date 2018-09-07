- Start Date: 2018-09-06
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

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
 
For symmetry, both of these APIs should be public. Making `modelFactoryFor` public would provide a hook
 that consumers can override should they desire to provide a custom `ModelClass` as an alternative
 to `DS.Model`.

## Detailed design

```typescript
class Store {
  modelFactoryFor(modelName: String): InstantiableModelClass {}
}
```

Due to previous complexity in the lookup of models in `ember-data`, we previously had both `modelFactoryFor`
and `_modelFactoryFor`. Despite the naming, both of these methods were private. During a recent cleanup phase,
we unified the methods into `_modelFactoryFor` and left a deprecation in `modelFactoryFor`. This RFC proposes
un-deprecating the `modelFactoryFor` method and making it public, while deprecating the private `_modelFactoryFor`.

More precisely:

- `store._modelFactoryFor` becomes deprecated and calls `store.modelFactoryFor`.
- `store.modelFactoryFor` becomes un-deprecated.

Users wishing to extend the behavior of `modelFactoryFor` could do so in the following manner:

**services/store.js**
```js
import Store from 'ember-data/store';

export default Store.extend({
  modelFactoryFor(modelName) {
    if (someCustomCondition) {
      return someCustomFactory;
    }
    
    return this._super(modelName);
  }
});
```

## How we teach this

This API would be intended for addon-authors and power users. It is not expected
that most apps would implement custom models, much as it is not expected that most
apps would implement custom `RecordData`. The teaching story would be limited to
documenting the nature and purpose of `modelFactoryFor`.

## Drawbacks

- Users may try to use the hook to instantiate records on their own. Ultimately, the store
  should still do the instantiating.
- We have not yet documented the APIs required to be implemented by a custom `ModelClass` and
  how/where a `record` accesses the backing `RecordData` instance.

## Alternatives

Users could define models in `models/*.js` that utilize a custom `ModelClass`.
However, such an API for custom classes would exclude the ability to dynamically
generate classes.

## Unresolved questions

None
