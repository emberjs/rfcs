- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Record Data Operations

## Summary

Introduces `RecordData.performMutation` which takes an `Operation` for local mutations
 of data. `Operations` are small, serializable objects describing a mutation.

## Motivation

- better organize API surface
- provide clear communication channels between primitives
- reduce API surface by disentangling DS.belongsTo / DS.hasMany from RecordData
- foundation for future operations based APIs (relationships, canonical state patches,
  logging, serialization, debugging APIs, reverse operations, transactions)
- improved factoring of internals
- easy deprecation path to singleton RD support

`Operations` are a foundational concept for [ACID](https://en.wikipedia.org/wiki/ACID_(computer_science)) transactions. Without the ability to
 describe an `operation` and a clear mental model of what operations exist and achieve,
 it is difficult to understand how an action affects state. Granularity and clarity is 
 **key**.

Let's take a look at a the [`RecordData`](https://github.com/emberjs/rfcs/pull/293)
methods that might better be understood as operations using [`identifiers`](https://github.com/emberjs/rfcs/pull/403).

**Methods That Mutate Record Properties**

```ts
interface RecordData {
  addToHasMany(key: string, recordDatas, idx: number) {}
  removeFromHasMany(key: string, recordDatas) {}
  removeFromInverseRelationships(isNew: boolean) {}
  setBelongsTo(key: string, recordData) {}
  setDirtyAttribute(key: string, value: any) {}
  setHasMany(key: string, recordDatas) {}
  rollbackAttributes() {}
}
```

**Methods That Mutate Record State**

```ts
interface RecordData {
  clientDidCreate() // "addRecord"
  // in the original RFC but we failed to implement (but is needed)
  clientDidDelete() // "removeRecord"
  unloadRecord() // "unloadRecord"
}
```

The "soup" of available methods prevents someone from knowing at a glance which methods
are for mutations, and whether these methods are for mutating local state, or communicating
remote state changes.

It also means that developers must memorize a large API surface area, and don't have a clear
picture of how these methods relate to each other in terms of what they achieve. Organizing
mutations as `operations` will help us provide both clearer mental model and a better decoupling.

### Eliminating unnecessary surface area and better decoupling

Implementations proving a path towards a nicer relationship graph have shown that the distinction
 between `belongsTo` and `hasMany` at the `RecordData` level is unnecessary, and only leads to
 extraneous method branching throughout the rest of `ember-data`'s primtives.
 
Furthermore, some `Record` and `RecordData` implementations may not provide relationships at all,
 leaving unnecessary dangling methods around to confuse folks exploring the codebase while debugging.

With these improvements to `RecordData` and associated changes to relationships and `Record` classes,
`ember-data` will no longer need to be coupled to the existing understanding or semantics of `belongsTo`
and `hasMany`, allowing custom records to define new types of relationships more easily (such as `map`,
`set`, `union`, `enum`) without shoehorning them through `attribute` methods or fudging them into `belongsTo`
or `hasMany` in form.

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

### Operations

**Operations That Mutate Record Properties**

**`addToHasMany`**

```ts
interface AddToRelatedRecordsOperation extends Operation {
  op: 'addToRelatedRecords';
  record: RecordIdentifier;
  property: string; // "relationship" propertyName
  value: RecordIdentifier; // related record
}
```

**`removeFromHasMany`**

```ts
interface RemoveFromRelatedRecordsOperation extends Operation {
  op: 'removeFromRelatedRecords';
  record: RecordIdentifier;
  property: string; // "relationship" propertyName
  value: RecordIdentifier; // related record
}
```

**`setBelongsTo`**

```ts
interface ReplaceRelatedRecordOperation extends Operation {
  op: 'replaceRelatedRecord';
  record: RecordIdentity;
  property: string;
  value: RecordIdentity;
}
```

**`setDirtyAttribute`**

```ts
interface ReplaceAttributeOperation extends Operation {
  op: 'replaceAttribute';
  record: RecordIdentifier;
  property: string; // "attr" propertyName
  value: any;
}
```

**`setHasMany`**

```ts
interface ReplaceRelatedRecordsOperation extends Operation {
  op: 'replaceRelatedRecords';
  record: RecordIdentity;
  property: string;
  value: RecordIdentity[];
}
```

**`rollbackAttributes`**

Once "logs" are present this could be modeled as a series of reverse operations.

```ts
interface RollbackAttributesOperation extends Operation {
  op: 'rollbackAttributes',
  record: RecordIdentity;
}
```

**Operations That Mutate Record State**

**`clientDidCreate`**

```ts
interface AddRecordOperation extends Operation {
  op: 'addRecord';
  record: Record;
}
```

**`clientDidDelete`**

```ts
interface RemoveRecordOperation extends Operation {
  op: 'removeRecord';
  record: Record;
}
```

**`unloadRecord`**

```ts
interface UnloadRecordOperation extends Operation {
  op: 'unloadRecord';
  record: Record;
}
```

### _initRecordCreateOptions

This API is problematic and hard to model, we need something, let's discuss.

### removeFromInverseRelationships

This API should be eliminated entirely, removeRecord and unloadRecord
 would take care of the use case *if* recordData tracks whether it is
 `isNew` and `isDeleted`.


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
