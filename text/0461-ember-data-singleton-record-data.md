- Start Date: 2019-03-06
- RFC PR: 461
- [RFC Tracking Issue #51](https://github.com/emberjs/rfc-tracking/issues/51)

# Singleton Record Data

## Summary

Ensures that `RecordData` can be implemented as a singleton, eliminates several redundant APIs
that creeped into the original implementation, and simplifies the method signatures of
RecordData APIs by using [`Identifiers`](https://github.com/emberjs/rfcs/pull/403).

## Motivation

`RecordData` is the data-cache primitive for information given to `ember-data`, and while
a cache-per-entity setup is sometimes desired, a singleton cache offers opportunities for
performance optimization and improved feature sets. Current default `RecordData`, and
`InternalModel` which it replaced are examples of cache-per-entity strategies.

Our original intent [when discussing `RecordData`](https://github.com/emberjs/rfcs/pull/293)
was for it to be possible to be implemented as a singleton; however, this intent was not
captured well in that RFC and while the APIs presented there would have enabled it, the
actual implementation differed in ways that prevent `singleton` implementations.

The introduction of `Identifiers` presents us with a good opportunity to refactor the API
surface of our cache-primitive, simplifying and streamlining how it works while solidifying
and codifying its ability to be a `singleton`.

### Reduce Overloading

Previously, to deal with the lack of a unified identifier concept, we overloaded many
`StoreWrapper` and `RecordData` method signatures with `modelName, id, clientId` as
`arguments`. The original intent was to overload **all** of their method signatures
in this way to ensure `singleton` implementations could be built, but we failed to
correctly implement the original `RecordData RFC` in this regard.

The introduction of `Identifiers` provides us a cleaner interface for communicating identity
in these APIs. For uniformity, methods will always take an `identifier` as their **first
argument**.

## Detailed design

Because `RecordData` is a public userland interface we
must rely on capabilities reporting to handle the
deprecations for it introduced by this RFC.

We will use this opportunity to improve the encapsulation
of `RecordData` via introduction of a sandbox.

This sandbox will be responsible for handling interop,
deprecations and other book-keeping tasks we require as needed
while ensuring that `RecordData` implementations are properly
encapsulated (e.g. implementations can only talk to other
implementations via public API).

RecordData implementations must now specify a `version`.
The implicit version when unspecified is `"1"` (the version
which this RFC supplants and deprecates). Implementations
without a version or with a version equal to `"1"` will
receive a deprecation notice the first time an instance of
a previously unseen `RecordData` class is encountered.

In keeping with the support policy of `Ember.js`, future
releases of `ember-data` will support the versions of `RecordData`
current at the time of the last two `ember-data` `LTS` releases.

The APIs proposed by this RFC constitute version `"2"`
and will be used by `ember-data` when the following is
`true`.

```js
recordData.version === "2";
```

It is possible that once this upgrade to `identifiers` has occurred that we never need to
increment this `version` again. If so, a future RFC may choose to deprecate the `version`
property altogether once "version 1" is no longer supported.

### Sandboxing

The sandbox implementation would guarantee the following

Beginning with this RFC, `StoreWrapper.recordDataFor` will no longer directly return the `RecordData`
instance provided by the `createRecordDataFor` hook, instead returning a delegate.

This encapsulates the `RecordData` feature ensuring that communication is via public API
and provides us the ability to manage the three concerns listed above and described below.

**Managing Deprecations**

When a `RecordData` method or method signature is deprecated, we will detect calls to deprecated
methods or calls using deprecated method signatures and print the appropriate deprecation messages.

**Version Interop**

When a `RecordData` method or method signature is deprecated the instance provided by `createRecordDataFor`
may not have implemented the deprecated method or signature. Using the supplied version, calling context,
and other information available we will transform calls to deprecated methods or signatures
into their supported equivalent.

**Updates to Store Methods**

<table>
<thead>
<tr>
<th>Old</th>
<th>New</th>
</tr>
</thead>
<tbody>
<tr>
<td width="50%" valign="top">

```ts
class Store {
  createRecordDataFor(
    modelName: string,
    id: string,
    clientId: string | null,
    storeWrapper
  ): RecordData;
  createRecordDataFor(
    modelName: string,
    id: null,
    clientId: string,
    storeWrapper
  ): RecordData {}
}
```

</td>
<td valign="top">

```ts
class Store {
  createRecordDataFor(identifier: RecordIdentifier, storeWrapper): RecordData {}
}
```

</td>
</tr>
</tbody>
</table>

**Updates to StoreWrapper Methods**

<table>
<thead>
<tr>
<th>Old</th>
<th>New</th>
</tr>
</thead>
<tbody>
<tr>
<td valign="top">

```ts
class RecordDataStoreWrapper {
  recordDataFor(
    modelName: string,
    id: string | null,
    clientId: string | null
  ): RecordData {}

  notifyPropertyChange(
    modelName: string,
    id: string | null,
    clientId: string | null,
    key: string
  ): void {}

  notifyHasManyChange(
    modelName: string,
    id: string | null,
    clientId: string | null,
    key: string
  ): void {}

  notifyBelongsToChange(
    modelName: string,
    id: string | null,
    clientId: string | null,
    key: string
  ): void {}

  attributesDefinitionFor(modelName: string): AttributesSchema {}

  relationshipsDefinitionFor(modelName: string): RelationshipsSchema {}

  setRecordId(modelName: string, id: string, clientId: string): void {}

  disconnectRecord(
    modelName: string,
    id: string | null,
    clientId: string | null
  ): void {}

  @deprecated // Use hasRecord
  isRecordInUse(
    modelName: string,
    id: string | null,
    clientId: string | null
  ): boolean {}

  inverseForRelationship(modelName: string, key: string): string {}
  inverseIsAsyncForRelationship(modelName: string, key: string): boolean {}
}
```

</td>
<td valign="top">

```ts
class RecordDataStoreWrapper {
  recordDataFor(identifier: RecordIdentifier): RecordData {}

  @deprecated // use notifyChange instead
  notifyPropertyChange(
    modelName: string,
    id: string | null,
    clientId: string | null,
    key: string
  ): void {}

  @deprecated // not in the original RFC, use notifyChange instead
  notifyHasManyChange(
    modelName: string,
    id: string | null,
    clientId: string | null,
    key: string
  ): void {}

  @deprecated // not in the original RFC, use notifyChange instead
  notifyBelongsToChange(
    modelName: string,
    id: string | null,
    clientId: string | null,
    key: string
  ): void {}

  // replaces notifyErrorsChange introduced by RecordData Errors RFC
  notifyChange(
    identifier: RecordIdentifier,
    namespace: "errors" | "relationships" | "attributes" | "meta" | "state"
  ): void {}

  attributesDefinitionFor(identifier: RecordIdentifier): AttributesSchema {}

  relationshipsDefinitionFor(
    identifier: RecordIdentifier
  ): RelationshipsSchema {}

  setRecordId(identifier: RecordIdentifier, id: string): void {}

  disconnectRecord(identifier: RecordIdentifier): void {}

  @deprecated // Use hasRecord
  isRecordInUse(
    modelName: string,
    id: string | null,
    clientId: string | null
  ): boolean {}

  hasRecord(identifier: RecordIdentifier): boolean {}

  inverseForRelationship(identifier: RecordIdentifier, key: string): string {}
  inverseIsAsyncForRelationship(
    identifier: RecordIdentifier,
    key: string
  ): boolean {}
}
```

</td>
</tr>

</tbody>
</table>

**Updates to RecordData Methods**

<table>
<thead>
<tr>
<th>Old</th>
<th>New</th>
</tr>
</thead>
<tbody>
<tr>
<td valign="top">

```ts
interface LegacyResourceIdentifierObject {
  type: string;
  id: string | null;
  clientId: string | null;
}

interface RecordDataV1 {
  unloadRecord(): void;

  @deprecated // This RFC deprecates this method without replacement. It was not in
  // the original RecordData RFC and after investigation is unneeded once the singleton
  // and identifiers refactoring are completed.
  isRecordInUse(): boolean;
  isEmpty(): boolean;
  isNew(): boolean;

  getAttr(propertyName: string): any;
  isAttrDirty(propertyName: string): boolean;
  changedAttributes(): AttributesChanges;
  hasChangedAttributes(): boolean;
  rollbackAttributes(): void;

  @deprecated // this RFC deprecates this method entirely
  getBelongsTo(propertyName: string): { data: LegacyResourceIdentifierObject };
  @deprecated // this RFC deprecates this method entirely
  getHasMany(propertyName: string): { data: LegacyResourceIdentifierObject[] };

  willCommit(): void;
  didCommit(data: any): void;

  @deprecated // this RFC deprecates this method entirely
  _initRecordCreateOptions(options: object): object;
  clientDidCreate(): void;

  @deprecated // this RFC deprecates this method entirely and without replacement
  getResourceIdentifier(): LegacyResourceIdentifierObject;

  // original RecordData RFC specified setBelongsTo
  // with the 2nd arg being LegacyResourceIdentifierObject
  // but that was not what the resulting implementation in
  // ember-data did
  setDirtyBelongsTo(propertyName: string, value: RecordData | null): void;

  // original RecordData RFC specified setAttribute
  // but that was not what the resulting implementation in
  // ember-data did
  setDirtyAttribute(propertyName: string, value: any): void;

  // original RecordData RFC specified setHasMany
  // with the 2nd arg being LegacyResourceIdentifierObject[]
  // but that was not what the resulting implementation in
  // ember-data did
  setDirtyHasMany(propertyName: string, value: RecordData[]): void;

  // original RecordData RFC specified addToHasMany
  // with the 2nd arg being LegacyResourceIdentifierObject[]
  // but that was not what the resulting implementation in
  // ember-data did
  addToHasMany(propertyName: string, values: RecordData[], startIndex?: number): void;

  // original RecordData RFC specified removeFromHasMany
  // with the 2nd arg being LegacyResourceIdentifierObject[]
  // but that was not what the resulting implementation in
  // ember-data did
  removeFromHasMany(propertyName: string, values: RecordData[]): void;

  @deprecated // this RFC deprecates this method entirely and without replacement
  removeFromInverseRelationships(isNew: boolean): void;
}
```

</td>
<td valign="top">

```ts
interface RecordDataV2 {
  version: "2";
  unloadRecord(identifier: Identifier);
  isEmpty(identifier: Identifier): boolean;
  isNew(identifier: Identifier): boolean;

  getAttr(identifier: Identifier, propertyName: string): any;
  isAttrDirty(identifier: Identifier, propertyName: string): boolean;
  changedAttrs(identifier: Identifier): AttributesChanges;
  hasChangedAttrs(identifier: Identifier): boolean;
  rollbackAttrs(identifier: Identifier): void;

  getRelationship(
    identifier: Identifier,
    propertyName: string
  ): { data: Identifier | Identifier[] };

  willCommit(identifier: Identifier): void;
  didCommit(identifier: Identifier, data: any);

  clientDidCreate(identifier: Identifier, options: object): void;

  setBelongsTo(
    identifier: Identifier,
    propertyName: string,
    value: Identifier | null
  ): void;

  setAttr(identifier: Identifier, propertyName: string, value: any): void;

  setHasMany(
    identifier: Identifier,
    propertyName: string,
    values: Identifier[]
  ): void;

  addToHasMany(
    identifier: Identifier,
    propertyName: string,
    values: Identifier[],
    startIndex?: number
  ): void;

  removeFromHasMany(
    identifier: Identifier,
    propertyName: string,
    values: Identifier[]
  ): void;
}
```

</td>
</tr>

</tbody>
</table>

## How we teach this

`RecordData` is primarily an API meant for power-user addon-authors,
and not something we expect everyday users of `ember-data` to be intimately
familiar with. It is unlikely that we produce a `guide` for using `RecordData`, but
a public `typescript` interface for `RecordData` should be introduced with
API Documentation for the available APIs and their roles.

## Drawbacks

It introduces churn in APIs we only recently introduced (in the past 6 months); however we have
strong reason to believe that very few implementations exist and that those which do have their
migration path covered by the sandboxed RecordData implementation.

## Alternatives

- keep the status quo: large number of arguments to methods, no support for singleton RecordData,
  renders `Identifier` RFC largely useless and makes future iteration on RecordData similarly difficult.
