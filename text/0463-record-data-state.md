---
Start Date: 2019-03-13
Relevant Team(s): data
RFC PR: https://github.com/emberjs/rfcs/pull/463
Tracking: https://github.com/emberjs/rfc-tracking/issues/47

---
# Record State on Record Data RFC
  

## Summary

This RFC is a follow-up RFC for #293 RecordData.

Add `isNew` , `isDeleted` and `isDeletionPersisted` methods on Record Data that expose information about newly created and deleted states on the record

## Motivation

RecordData handles managing local and server state for records. While the current API surface of RecordData handles most data wrangling needs, as we have used RD internally in ED and in addons we have repeatedly run into the need of knowing whether a record is in a new or a deleted state.  The initial version of Record Data made the choice to not expose state information in order to limit the scope of the RFC and to give us time to come up with a design.

## Detailed design

Add the following methods on the RecordData interface:

```ts    
interface RecordData {
    
  // To be added in this RFC
  isNew(identifier: RecordIdentifier): boolean

  isDeleted(identifier: RecordIdentifier): boolean

  isDeletionCommitted(identifier: RecordIdentifier): boolean
    
  setIsDeleted(identifier: RecordIdentifier, boolean: isDeleted): void
}
```

`isNew` should return true if and only if the record with the given identifier was created on the client side (via clientDidCreate) and not successfully committed on the server, otherwise false.

`isDeleted` should return true if and only if the record has been marked for deletion on the client side, otherwise false

`isDeletionPersisted` should return true if and only if the record had been marked for deletion and that deletion has been successfully persisted, otherwise false.

The TS interface for the all of Record Data with the new methods can be be found [here](https://github.com/emberjs/data/blob/igor/record-data-state-interface/packages/store/addon/-private/ts-interfaces/record-data.ts#L13).

In order to notify changes to the state flags we would also add a notification method to the store wrapper which is passed to RecordData:

```ts
interface RecordDataStoreWrapper {
  notifyStateChange(identifier: RecordIdentifier, key: 'isNew' | 'isDeleted' | 'isDeletionPersisted');
}
```

Which would bring Record Data Store Wrapper to look like:

```ts
export interface RecordDataStoreWrapper {
  // This rfc
  notifyStateChange(identifier: RecordIdentifier, key: 'isNew' | 'isDeleted' | 'isDeletionPersisted');

  // Existing
  notifyAttributeChange(identifier: RecordIdentifier, key: string);
  notifyRelationshipChange(identifier: RecordIdentifier, key: string);
  notifyErrorsChange(identifier: RecordIdentifier, key: string);

  relationshipsDefinitionFor(identifier: RecordIdentifier): RelationshipsSchema
  attributesDefinitionFor(identifier: RecordIdentifier): AttributesSchema
  setRecordId(identifier: RecordIdentifier, id: string);
  disconnectRecord(identifier: RecordIdentifier);
  isRecordInUse(identifier: RecordIdentifier): boolean;
}
```

Currently calling `rollbackAttributes` rolls back `isDeleted` to a non deleted state. This logic would be the responsibility of Record Data to implement. 

## How we teach this
We currently do not have a comprehensive way to teach RecordData api. This RFC will be tought alongisde the rest of upcoming Record Data docs. 


## Alternatives

Instead of separate methods, have a single methods, somehting like `getState` that returns a POJO with keys like 

    { isDeleted: true, isNew: false }