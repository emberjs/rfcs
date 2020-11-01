---
Start Date: 2019-03-13
Relevant Team(s): data
RFC PR: https://github.com/emberjs/rfcs/pull/465
Tracking: https://github.com/emberjs/rfc-tracking/issues/46

---
# Record Data Errors RFC

## Summary

This RFC is a follow-up RFC for #293 RecordData.

Exposes the content of Invalid Errors returned by the adapter on Record Data.

## Motivation

Currently Record Data manages and exposes all of the attributes and relationships for a record. However, the initial version of Record Data made the choice to not expose error information in order to limit the scope of the RFC and to give us time to come up with a design. This RFC, together with the Request State RFC, addresses this capability gap.

When a user sends a record save request, it can fail in two different ways:

- It can fail as a generic Adapter Error, and put the record in an `isError` state. This corresponds to any failure, including those such as the network being down, server returning 500s or the auth layer returning a 401. Because these errors are not tied to the Record or it's data, they do not belong in the Record Data layer and will be exposed separately as part of the Request State RFC
- However, the request can also fail with a more specific `InvalidError` This currently puts the record in the `invalid` state. `InvalidError` corresponds to a specific validation failure of the request being made. Currently, the default adapter implementation creates an `InvalidError` if the server returns a `422` . `InvalidError` payload follows the JSON API error object spec, and if the error payload contains pointers those get mapped to attributes on a record. Because an Invalid Error maps to the data on a record, it should be managed by Record Data together with the attributes and relationships.

## Detailed design

Currently on a failed save, Record Data receives a call to 

`commitWasRejected(recordIdentifier: RecordIdentifier): void;`

This RFC proposes passing an optional errors object that would follow a subset of the JSON api errors spec, only in case an invalid error has been returned. We would also expose a getter for the errors.

```ts
interface RecordValidationError {
  title: string;
  detail: string;
  source: {
    pointer: string;  <relative to record>
  }
}

interface RecordData {
  commitWasRejected(recordIdentifier: RecordIdentifier, errors?: RecordValidationError[]): void;
  getErrors(recordIdentifier: RecordIdentifier): RecordValidationError[]
}
```

`RecordValidationError` follows the subset of the JSON api errors spec. For example, if the record being saved was rejected because the attribute `password` was empty, the `RecordValidationError` could look like: 

```ts
{
  title: 'Field cant be empty',
  detail: 'Field must be at least 8 characters long',
  source: {
    pointer: 'attributes/password'
  }
}
```

The source pointer is a JSON pointer relative to the Resource Object.   

We would also add a method on the `RecordDataStoreWrapper` to enable Record Data to notify the store that the errors properties have changed.
```ts
class RecordDataStoreWrapper {
  notifyErrorsChange(recordIdentifier: RecordIdentifier)
}
```

There would be no api for changing the errors from the client side, they would be read only from the perspective of the `DS.Model`  Currently `DS.Model`  exposes an `Errors` object, which has the behavior of removing errors if an attribute has been modified. This behavior would not be the responsibility of the `RecordData` layer which is tasked only with exposing the errors it received from the server and would remain an implementation detail of `DS.Model`

## How we teach this

We currently do not have a comprehensive way to teach the RecordData api. This RFC will be taught alongisde the rest of upcoming Record Data docs. 

## Alternatives

We could add a method to `RecordData` to make the errors setable from the model, something like `setErrors`. However, conceptually, because updated errors don't get sent back to the server they shouldn't be managed by the data cache layer. Removing or updating Error values should be the responsibility of the Model or UI layer.

