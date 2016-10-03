- Start Date: 2016-10-02
- RFC PR:
- Ember Issue:

# Summary

Add support for saving a record and nested records within the same request.

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


# Detailed design

## Overview

There are two main things the implementation needs to concern itself with.

  1. Handling the save lifecycle for saved nested objects and
  2. Assigning ids to saved nested objects during create.

The nested object lifecycle is managed by entangling the nested object with the
promise for saving the parent object.

Assigning ids from the server is handled by having the store generate client ids
when the child objects are entangled with their parent object's save promise,
and then having those client ids added to the child object's payload within
`included`, during `normalizePayload`.

In the event that the server echoes back the client ids, users will not need to
add anything to `normalizePayload`.  To handle cases where the server does not
do this, `normalizePayload` will be passed a snapshot of the parent object, and
an array of snapshots of the saved nested objects, so that it can find their
client ids and add them to their respective structures within the `included`
portion of the payload.

## Example Usage with API Server Echoing clientId

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

## Example Usage with API Server Not Echoing clientId


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
                    snapshot,
                    nestedSaves) {

    let jsonApiPayload = this.super(...arguments);
    nestedSaves.forEach((childSnapshot) => {
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

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

How should this feature be introduced and taught to existing Ember
users?


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

# Unresolved questions

Are there issues with transitioning nested saved objects to an invalid state
even in cases where the nested object itself did not have errors?

