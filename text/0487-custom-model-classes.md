- Start Date: 2019-05-09
- Relevant Team(s): data 
- RFC PR: [PR](https://github.com/emberjs/rfcs/pull/487)
- Tracking: [Tracking](https://github.com/emberjs/rfc-tracking/issues/53)

# Custom Model Class RFC

## Summary

This RFC is a follow-up RFC for #293 RecordData and replaces [https://github.com/runspired/rfcs/blob/ember-data-custom-records/text/0000-ember-data-model-factory-for.md#ember-data--modelfactoryfor](https://github.com/runspired/rfcs/blob/ember-data-custom-records/text/0000-ember-data-model-factory-for.md#ember-data--modelfactoryfor)

Create a way for addons and user to define their own implementation of a model class. Adds an `instantiateRecord` method to `Store` which would allow addons and ED itself to offer a clean replacement the default DS.Model with a custom Model class. 

## Motivation

- Allowing Addons to experiment with replacing the default DS.Model implementation ([https://github.com/runspired/rfcs/blob/ember-data-custom-records/text/0000-ember-data-model-factory-for.md#ember-data--modelfactoryfor](https://github.com/runspired/rfcs/blob/ember-data-custom-records/text/0000-ember-data-model-factory-for.md#ember-data--modelfactoryfor))
- Currently Ember Data conflates schema information and record implementation together as part of properties defined on `DS.Model`.
DS.Model both defines the schema, that is the attributes and relationships present on a type, and the model implementation, that is the
actual class which is exposed to the app containing those properties. This has several disadvantages:
- it makes it hard to optimize the schema information, and ties to a definition of a JS class
- it makes it hard to statically analyze the schema
- it requires a 1-1 correspondence between the number of backing types in the system and the number of model classes, resulting
in an app sometimes having to ship a much higher number of models than it otherwise would have.

This RFC represents the first step in separating the record implementation from schema information.

## Detailed design

After the work in [Record Data Errors RFC](https://github.com/emberjs/rfcs/pull/465),  [Record Data State RFC](https://github.com/emberjs/rfcs/pull/463), and [Request State Service RFC](https://github.com/emberjs/rfcs/pull/466) we have enough coverage in public apis that we can implement the entire DS.Model class with just a few changes to Store APIs. 

We will need five separate changes: 

- A way to instantiate custom Records and for them to be able to access underlying record data
- A way for those records to be notified of changes in Record Data
- A way to access schema information from the store and not from the DS.Model static properties
- Replacement of a few store methods that take DS.Models
- Deprecate passing DS.Model classes to adapter/serializers/exposing them on Snapshots

## Instantiating and destroying custom Records

We need to make the following changes:

* expose a method on the store for instantiating a record
* add a hook to be notified when a record is being destroyed for potential cleanup
* a way for the record to get notified when RecordData values change

The following interface shows the interface of these methods:

```ts
interface RecordDataWrapper { // proxies a subset of RD methods and hides the rest
  getAttr(identifier: RecordIdentifier, key: string)
  isAttrDirty(identifier: RecordIdentifier, key)
    
  changedAttributes(identifier: RecordIdentifier)
  hasChangedAttributes(identifier: RecordIdentifier)
  rollbackAttributes(identifier: RecordIdentifier)
  
  getRelationship(identifier: RecordIdentifier, key: string)
  
  setRecordId(identifier: RecordIdentifier, id: string): void;
  
  getErrors(identifier: RecordIdentifier)
  getMeta(identifier: RecordIdentifier)
  
  isNew(identifier: RecordIdentifier): boolean;
  isDeleted(identifier: RecordIdentifier): boolean;

  setDirtyAttribute(identifier, key: string, value: any): void;

  addToHasMany(key: string, identifiers: Identifier[], idx?: number): void;
  removeFromHasMany(key: string, identifiers: Identifier[]): void;
  setDirtyHasMany(key: string, identifiers: Identifier[]): void;

  setDirtyBelongsTo(name: string, identifier: Identifier | null): void;
}
  
interface RecordDataFor {
  (identifier: RecordIdentifier): RecordDataWrapper
}
    
class Store {
  instantiateRecord(
    identifier: RecordIdentifier, 
    createRecordArgs: { [key: string]: any }, // args passed in to store.createRecord() and processed by recordData to be set on creation
    recordDataFor: RecordDataFor, 
    notificationManager: NotificationManager): unknown
    
  teardownRecord(record): void
}
```

Instead of passing the entire RecordData Class to `instantiateRecord`, we pass in a lookup method that returns a record data wrapper which exposes only the local facing methods and hide the server/adapter facing methods that the Model should not have access to.

We also expose `teardownRecord`, which will get called whenever a record is getting unloaded, or otherwise disposed of (if we add future ways of destroying records that are uncoupled from unloading) from the identity map, thus giving addons an opportunity to perform cleanup. 

## Change Notification

In order for the record class to learn about changes to the underlying data, it will be passed a `NotificationManager` as a constructor argument, which will allow it to
subscribe and unsubscribe to notifications of underlying changes to the data. Once `RecordData` calls one of the notification methods, the notification manager will call 
any registered callback for the given identifier, and pass in the type of the notification, allowing the record the opportunity to repull the data if needed. There are no guarantees around the timing of the notification callback being called after `RecordData` informs the store of changes. We expect that in a modern Ember app with tracked properties, this wouldn't be the common path for tracking changes. 

```ts
function unsubscribe(token: UnsubscribeToken)

interface NotificationCallback {
  (identifier: Identifier, notificationType: 'attributes' | 'relationships' | 'errors'  | 'meta'  | 'unload'): void;
}
    
interface NotificationManager {
  subscribe(identifier, NotificationCallback): UnsubscribeToken
 }
```

## Exposing schema information

We currently keep schema information on `DS.Model` class. In order to allow for custom Model implementations we need to allow lookup of schema info from the store. We already have specified a schema api that RecordData consumes: `attributesDefinitionFor` and `relationshipDefinitionFor`. We would define on a schema interface that the store would expose and addons could use. **The schema methods are not ergonomic on purpose.** They match the current Record Data apis and are designed as a stepping stone on the way of having a better, user facing schema APIs. Addons could provide their own `SchemaDefinitionService` by calling `registerSchemaDefinitionService`. We would initially not allow calling `registerSchemaDefinitionService` more than once, but this constraint could potentially be relaxed in the future. The schema info is currently primarily geared towards being used by the internal relationship handling layer, the serializer/adapter layers and the DebugAdapter. Schema methods support both static and dynamic schema computation. For static schemas, the method can always respond with a schema definition based on the type passed, and for dynamic changing schemas, it can look up the underlying data by the identifer which is passed in. We would also add a method called `doesTypeExist`, which would return `true` if ED knew that a given string is a model type and `false` otherwise.
  

```ts
export interface RelationshipDefinition {
  kind: 'hasMany'| 'belongsTo';
  type: string;
  options: { [key: string]: any } ;
  name: string;
}
    
interface RelationshipsDefinition {
  [key: string]: RelationshipDefinition
}
    
interface AttributeDefinition {
  name: string;
  options: { [key: string]: any };
  type: string;
}

interface AttributesDefinition {
  [key: string]: AttributeDefinition
}
    
interface SchemaDefinitionService {
  // Following the existing RD implementation 
  attributesDefinitonFor(identifier: RecordIdentifier | type: string): AttributesDefiniton
  
  // Following the existing RD implementation
  relationshipsDefinitionFor(identifier: RecordIdentifier | type: string): RelationshipsDefinition

  doesTypeExist(type: string): boolean
}

class Store {
  registerSchemaDefinitionService(schema: SchemaDefinitionService): void

  getSchemaDefinitionService(): SchemaDefinitionService
}
```

## Adding store methods for manipulating records 

Currently there exist methods on DS.Model that call into `internalModel` for it's functionality. In order for a parallel implementation
to be possible, we need to expose that functionality through public methods on the store.

```ts
class Store {
  saveRecord(record): Promise // equivalent of currently doing record.save()
  serializeRecord(record): any // equivalent of currently doing record.serialize()
  relationshipReferenceFor(identifier: RecordIdentifier, key: string): RelationshipReference
}
```

this would allow you to have a custom model class like this: 

```ts
class CustomModel {
  save() {
    return this._store.saveRecord(this);
  }
}
```

### Record Arrays

Currently Ember Data manages live Record Arrays which are returned as a response to query methods such as `findAll` or `queryRecords`. Because ED can track records which are returned from `instantiateRecord`, it will be able to seamlessly manage custom models which are part of Ember Data record arrays and clean them up correctly. 

If an addon implements it's own record array like structures, it will be able to manage membership of Ember Data default records by subclassing `teardownRecord`, which gives it a convinient place to listen for a record being destroyed.


## Deprecating DS.Model being passed to serializers and adapters and store.modelFor

Currently some adapter/serializer methods get the underlying class passed in. For example:

```ts
// Adapter
createRecord(store, type, snapshot)
findRecord(store, type, id, snapshot)

// Serializer
normalizeResponse(store, primaryModelClass, payload, id, requestType)

// Store
modelFor(modelName)
```

### Serializing

When serializing we already have a `Snapshot` passed in as an argument which has all of the information that a `ModelClass` provides. We would deprecate the class argument being passed in, and instruct the user to refactor towards using the `Snapshot`.

For example: 
```ts
createRecord(store, type, snapshot) {
  let url = `/api/${type.modelName}`;
}
```

would become

```ts
createRecord(store, type, snapshot) {
  let url = `/api/${snapshot.modelName}`;
}
```

 For backwards compatibility, we would still lookup the class from the registry, and if we found a class we would return it but would no longer error if null was returned. If we did not find the class, we would create a shim class that exposed a deprecated `modelName`. We would deprecate accessing the properties on the class when passed to serializer/adapter by wrapping it in a proxy in dev mode. Currently Snapshots contain all of the data that is available on the Model class which would be needed for serializing/normalizing.

### Normalizing

While `Snapshot` roughly corresponds to a `Request` like object to be used while serializing, and we intend to in the future refactor it to be a request object, we don't have a similar construct when normalizing. In the future, normalizing will have a corresponding `Response` like object exposed, but for the time being we can still deprecate using the `primaryModelClass` argument for anything other than `modelName`, `eachAttribute`, `eachRelationship`. If a user accesses any other property on the `primaryModelClass` they will receive a deprecation.

If there was a custom model instance provided, and we had no corresponding class to pass in to `normalizeResponse` we would pass in a shim class that only exposed the `modelName` property, and `eachAttribute` and `eachRelationship` methods which would proxy to the underlying schema methods, thus allowing the normalization layers to continue working with custom model classes.
 
```ts
interface ClassSchemaShim {
  modelName: string;
  eachAttribute( (name: string, attr: AttributeDefinition): void ): void;
  eachRelationship( (name: string, attr: AttributeDefinition): void ): void;
}
```


## How we teach this

This is a very addon/very power user specific api, and would be inappropriate for the default guides. We would document it in the Api docs and potentially if there was a guide for Ember Data addon developers.

