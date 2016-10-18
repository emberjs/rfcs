- Start Date: 2016-10-02
- RFC PR:
- Ember Issue:

# Summary

Support accurate state tracking and record matching when
saving a record and nested records within the same request.

This rfc proposes two main API changes:

  1. Add a `record.savedWith` method
  To be used like:
  ```js
    let promise = parentRecord.save();
    childRecord.savedWith(promise):
  ```
  where the childRecord will now follow the state transitions of the parent record.

  2. Allow the `store.push` method to receive an optional client id, which it will
  use as a fallback to avoid creating duplicate records.

# Motivation

Apps commonly want to save nested structures.  This can happen, for instance, on
a page for creating an object in which child objects are created at the same
time.  Usually the motivations are twofold: performance and transactional saves.

Achieving this today has problems both in the create and update case.  The
create case requires manually transitioning the child objects, or discarding
them, both of which are undesirable.

The update case is achieved today using EmbeddedRecordsMixin, but the child
objects will not go through the normal save lifecycle, and so miss out on
correctly dirty tracking &c.

There have been several issues and PRs filed to address this problem,
some are:
https://github.com/emberjs/data/pull/4441
https://github.com/emberjs/data/issues/1829

There is also an Ember Data addon which attempts to solve this problem:
https://www.npmjs.com/package/ember-data-save-relationships

-- In general, the existing addons and solutions have been great as workarounds
for people, but have not presented a cohesive and stable story for managing nested saves.
in Ember Data.

-- Another problem which has often been brought up in Ember Data is the race condition
when receiving out of band updates on record creation.

When you save a newly created record, and if you receive an out of band web socket update,
before the save has returned, you will end up with two ember data records.
```js
  let user1 = this.store.createRecord('user');
  user1.save();
  // out of band web socket update comes and tells you have a new user on the server
  let user2 =  store.push({ id:1, type: 'user'});
  // now save comes back and tells you the newly assigned id to user1 is '1'
  // it turns out that the out of band update was for the newly created user, and now
  // you have two user objects with same id
```
clientId support offers a mitigation strategy for this problem, because, if the clientId is
reflected back in the web socket push, we can tell that the incoming record is the same
as the inflight one.

# Detailed design

## Overview

There are two main things the implementation needs to concern itself with.

  1. Handling the save lifecycle for saved nested objects and
  2. Assigning ids to saved nested objects during create.
  3. Convenience methods for serializers

## 1. Handling the save lifecycle for saved nested objects

Currently, even if your serializer saves multiple records, it is very hard to/impossible
manage the lifecycle of the records being bulk saved.

This RFC proposes an API for entangling nested objects with the
promise for saving the parent object.

### APIs
- Add the method `DS.Model#savedWith`

`DS.Model#savedWith(promise)` is passed the `promise` for an ancestor object's
save.  It indicates that this record is being saved within the same promise.
When invoked, it will transition the record to `inFlight`.

`savedWith` returns a promise for the child record.

If `promise` rejects, the record will transition to an error state, depending on
what type of error the promise rejects with, and the childPromise will reject with
the error.

If `promise` fulfills, the record will transition to the saved state, and the child promise
will fulfill with the child record.

## 2. Assigning ids to saved nested objects during create.

Currently, it is impossible to match incoming records which were newly saved, with
records that exists in Ember Data store. Consider the problem of saving multiple records
together in an embedded structure, for example saving an order with multiple items in
an online shopping portal.

```js
order.save();
// results in:
{
  type: 'order',
  items: [{ type: 'item', name: 'laptop' },
    { type:'item', name: 'camera' }]
}
```

Currently it is impossible to easily match up returning items to existing records in the
ember data store.

The canonical way of solving this problem is by using client ids, which the server reflects
back to the client. However, even if you do this, there are no easy ways of matching the records
up in ember data.

We are proposing Ember Data add support for client ids, such that if the request returns back
client ids with records, the store will do the matching.

### APIs

- Add the getter `DS.Snapshot#clientId`
It is lazily generated when accessed and persisted locally.

- If the record resource returned from store.push is not already in the identity map,
but has a clientId, the store will try to look up the record by the clientId before pushing
a new record to the identity map.

For example, if you sent to the server

```js
// results in:
{
  type: 'order',
  items: [{ clientId: 1, type: 'item', name: 'laptop' },
    { clientId:2, type:'item', name: 'camera' }]
}
```

and the Serializer returns

```js
{
  data: {
    id: 1,
    type: 'order',
  },
  included: [
    { id:1, clientId: 1, type: 'item', name: 'laptop' }
    { id:2, clientId:2, type:'item', name: 'camera' }
  ]
}
```

The store would match up the records by the clientIds.

## 3. Convenience methods for serializers

In the event that the server echoes back the client ids, users will not need to
do anything. To handle cases where the server does not
do this, `normalizePayload` will be passed a snapshot of the parent object, and
an array of snapshots of the saved nested objects, so that it can find their
client ids and add them to their respective structures within the `included`
portion of the payload.

### APIs

- Add the parameter `snapshot` to `DS.Serializer#normalizePaylaod`
- Add the property `savedWith` to `DS.Snapshot`

## Examples

### Example Usage with API Server Echoing clientId

Here is an example of nested saving new objects:

```js
// models/order.js
DS.Model.extend({
  items: DS.hasMany('items'),
});

// routes/order.js
Ember.Route.extend({
  actions: {
    saveOrder(order) {
      let orderSavePromise = order.save();
      order.get('items').forEach(item => item.savedWith(orderSavePromise))
    }
  }
});
```

### Example Usage with API Server Not Echoing clientId


```js
// models/order.js
//  same as previous example

// routes/order.js
//  same as previous example

// serializers/order.js
JsonApiSerializer.extend({
  normalizeResponse(store,
                    primaryModelClass,
                    payload,
                    id,
                    requestType,
                    snapshot) {

    let jsonApiPayload = this.super(...arguments);
    snapshot.get('savedWith').forEach((childSnapshot) => {
      let nestedSavePayload =
        jsonApiPayload.included.find((childPayload) => {
          /*
            App specific logic for finding the included payload for this nested
            save object.
          */
        });

      nestedSavePayload.clientId = childSnapshot.clientId;
    });

    return jsonApiPayload;
  }
});
```

## Matching From `nestedSaves` vs Matching From `snapshot`

## Nesting Depth > 1

Nothing changes if saving a nested structure in which nesting occurs at depth
greater than 1 (eg if saving an object, its children, and grandchildren).  It
will simply be the case that more objects need to have their `savedWith` methods
called, those same objects will be added to `nestedSaves`, and they will need to
be included in the `included` section of the JSON API response payload.

Similarly, saving a nested structure in which some levels are skipped works.
For instance, saving a grandparent and grandchild, but not the intermediate
child, would result in the grandparent and grandchild being persisted and the
relationships set up but with an as-yet unpersisted child.  Supporting this case
is not a goal of this rfc, and I am not aware of any sensible use case for it,
but noting that it is supported is included here for completeness.

## Errors

If the adapter save promise rejects, all saved objects (ie the primary saved
object and the nested saved objects) transition to an error state.  The specific
error state depends on the specific type of error the promise rejected with, but
all saved objects transition to the same state.  In particular, they will all
transition to an invalid state if the promise rejects with an `InvalidError`
even though the errors returned from `extractErrors` may not include errors for
each of the saved objects.

For consistency, `snapshot` and `nestedSaves` will also be passed to
`extractErrors`.

# How We Teach This

The additional functionality proposed in this rfc is quite limited in scope.
All that should be needed is a guide exploring how to support nested saves, and
the additional API added to any discussion of implementing a custom serializer.

# Drawbacks

One obvious drawback is that this rfc introduces a new API, `savedWith`, and
adds parameters to `normalizePayload` and `extractErrors`.  Although this
doesn't add *much* to the API surface area, all such complexity comes at some
cost.

A second potential objection may be that the rfc feels a bit scenario solving,
albeit for a very common scenario.


# Alternatives

Other possibilities include:

  - Adding support for nested saving as part of a broader save coalescing API,
    similar to existing find coalescing and
  - Making the state transition API public.

Related work:

- [Discarding nested saved objects](https://github.com/emberjs/data/pull/4441)
- [ember-data-save-relationships](https://github.com/frank06/ember-data-save-relationships)
  (handles only the case where the server echoes the client id)

# Unresolved questions

Are there issues with transitioning nested saved objects to an invalid state
even in cases where the nested object itself did not have errors?

TODO: Improve names.

