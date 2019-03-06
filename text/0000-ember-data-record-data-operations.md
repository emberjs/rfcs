- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Record Data Operations

## Summary

Introduces `RecordData.performMutation` which takes an `Operation` for local mutations of
data. `Operations` are small, serializable objects describing a mutation.

## Motivation

- better organize API surface
- provide clear communication channels between primitives
- reduce API surface by disentangling DS.belongsTo / DS.hasMany from RecordData
- foundation for future operations based APIs (relationships, canonical state patches,
  logging, serialization, debugging APIs, reverse operations, transactions)
- improved factoring of internals
- easy deprecation path to singleton RD support

## Detailed design

- synchronous, because `setters` need to complete for "sync get"
- single operations, not batches, as "transactions" or "transforms" (orbit) would be a
  higher level primitive introduced lated if desired at a different position within the
  architecture
- op-codes are stable for serializability
- operations don't have identifiers, "transactions" or "transforms" would
- operations that fail should throw
- it is valid for a RecordData implementation to ignore an operation
  (but we should probably encourage logging ignored operations)
- operations are only for mutations of local state (at this time), use `push` for canonical
  _note: it is likely that we want operations within `push` for describing partial updates
  and a `relationships` RFC in particular will likely desire this and discuss it if so_

```typescript
export interface Operation {
  // the operation to perform
  op: string;
  // the record to be operated on
  record: RecordIdentifier;
  // the name of the property on the record to be mutated (if any)
  property?: string;
  // the new value for the property on the record (if any)
  // if this key is present, this value can be anything but `undefined`
  value?: Identifier | null | array | string | object | number | boolean;
}

export default interface RecordData {
    performMutation(operation: Operation): void
}
```

Example operations

```ts
interface ReplaceAttributeOperation extends Operation {
  op: 'replaceAttribute';
  record: RecordIdentifier;
  property: string; // "attr" propertyName
  value: any;
}
interface AddToRelatedRecordsOperation extends Operation {
  op: 'addToRelatedRecords';
  record: RecordIdentifier;
  property: string; // "relationship" propertyName
  value: RecordIdentifier; // related record
}
interface RemoveFromRelatedRecordsOperation extends Operation {
  op: 'removeFromRelatedRecords';
  record: RecordIdentifier;
  property: string; // "relationship" propertyName
  value: RecordIdentifier; // related record
}
```

### Existing APIs this would supercede

```ts
interface RecordData {
  setDirtyAttribute(key, value)
  addToHasMany(key, recordDatas, index)
  removeFromHasMany(key, recordDatas)
  setHasMany(key, recordDatas)
  setBelongsTo(key, recordData)
  removeFromInverseRelationships(isNew)
  rollbackAttributes()
  clientDidCreate()
}
```

Additionally we may consider modeling these APIs as operations as well

```ts
  didDelete() // cannonical update
  didCommit(data) // cannonical update, see also willCommit
  _initRecordCreateOptions(options) // 2nd phase of create, paired with clientDidCreate
  reset() // similar to rollbackAttributes
```

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
