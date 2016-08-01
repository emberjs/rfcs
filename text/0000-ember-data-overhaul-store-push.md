# ember-data-overhaul-push

- Start Date: 2016-08-01
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

A new `store.normalizePayload(modelName, payload)` is added which normalizes a
payload into JSON-API format. This normalized payload can then be used with
`store.push()` or the `push()` methods on the `BelongsTo` and `HasMany`
references.

Add a `store.pushRef()` which takes a JSON-API document and returns a
`RecordReference` or `RecordArrayReference` (proposed in
[RFC#160](https://github.com/emberjs/rfcs/pull/160)), depending on if the
primary data is a single resource or an array. References are returned here,
because they give access to `meta` and `links`.

# Motivation

## Status quo

Consider an application gets the data not only from the adapter, but also from
a different source (WebSockets for example). Ember Data has a
`store.pushPayload()` method, which allows to push records into the store
outside of adapters. It does however - compared to `store.push()` - not return
the pushed record(s).

The issues with `store.push` and `store.pushPayload` can be summarized as
follows:

- `store.pushPayload` doesn't return the pushed records (`store.push` does)
- `store.pushPayload` calls `serializer.pushPayload` which invokes
  `store.push`, though pushing data into the store shouldn't be the concern of
  the `serializer`
- `store.push` and `store.pushPayload` don't support `meta` and `links`
- the `modelName` passed to `store.pushPayload` is not passed explicitly to the
  serializers' `pushPayload`, making it difficult for the serializer to know
  what the primary data is (implicitly this is known by checking which
  serializer is invoked)

Also `BelongsToReference#push()` and `HasManyReference#push()` expect a
JSON-API document, but there is currently no easy way to normalize a payload
into the expected format:

- `serializer.normalizeResponse` is cumbersome, as it takes 5 parameters
- as stated in the documentation,
  "[`store.normalize`](http://emberjs.com/api/data/classes/DS.Store.html#method_normalize)
  converts a JSON payload into the normalized form that `push` expects", but it
  uses `serializer.normalize`, which only normalizes a single object
  - since the documentation is misleading, this seems like a bug
    ([#4285](https://github.com/emberjs/data/issues/4285))
  - this would be a good name for a method which normalizes a payload, but it's
    not clear how it can be refactored to support whole payload


The [`ds-pushpayload-return`](https://github.com/emberjs/data/pull/4110)
feature does address the problem of getting the pushed record(s), but there are
some issues:

- lock down the API for `store.pushPayload` to always return a materialized
  record(s)
- once locked down, we cannot refactor for lazy materialization (started in
  [PR#4272](https://github.com/emberjs/data/pull/4272))
- apps which heavily use `pushPayload` will then not profit from this

## Proposed solution

- add `store.normalizePayload` which normalizes a payload into JSON-API format
- add `store.pushRef` which pushes records into the store without
  materialization

# Detailed design

## `store.normalizePayload`

Add a `store.normalizePayload` method which takes an optional `modelName`
indicating the primary model (and hence which serializer should be used) and
the payload which should be pushed:

``` js
/**
  Normalize a payload into a JSON-API document, using the
  optional modelName as serializer for the primary data.

  If `modelName` is omitted, the application serializer is
  used. The `modelName` is passed down to the serializers'
  `normalizePayload` method - this can be used to determine
  what the primary data of the normalized payload is.

  @param modelName {String} optional modelName, indicating the
                            primary model of the payload
  @param payload {Object} payload which should be normalized
*/
normalizePayload(modelName, payload) {}
```

If a serializer implements `pushPayload`, then the normalizing-a-payload
functionality is basically already there. The problem is that the `serializer`
is expected to push the data into the store directly via `store.push`, instead
of returning the normalized payload. We can make use of this and re-use the
existing `pushPayload` on the serializer to provide a `normalizePayload` out of
the box, without immediate actions being required by app and addon authors.

_:zap: WARNING :boom: hacks ahead :boom: WARNING :zap:_

Given this implementation of `normalizePayload` on the store:

``` js
DS.Store.reopen({

  normalizePayload(modelName, payload) {
    let serializer = this.serializerFor(modelName);

    if (serializer.normalizePayload) {
      return serializer.normalizePayload(this, ...arguments);
    }

    if (serializer.pushPayload) {
      deprecate("rename `serializer.pushPayload` to `serializer.normalizePayload`" +
                " and return normalized payload instead of pushing it into the store");
      return serializer.__normalizePayload(this, payload);
    }

    assert("serializer needs to implement normalizePayload");
  }

})
```

Then the `DS.Serializer` can be adapted as follows to re-use the existing
payload-normalization behavior:

``` js
DS.Serializer.reopen({

  __normalizePayload(store, payload) {
    const originalPushMethod = store.push;
    let normalizedPayload;

    // temporarily patch store.push, which is invoked in
    // serializer.pushPayload
    store.push = function(_normalizedPayload) {
      normalizedPayload = _normalizedPayload;
    }

    this.pushPayload(store, payload);

    // restore original store.push
    store.push = originalPushMethod;

    return normalizedPayload;
  }

})
```

Drawback: this only works if `store.push` is used only once within
`serializer.pushPayload` though. A check could be added to see if `store.push`
is invoked multiple times within `serializer.pushPayload`. If this is the case,
then we can't automatically and reliably infer the
`serializer.normalizePayload` behavior.

The documentation for `DS.Serializer` is updated as follows:

``` js
/**
  Normalize a payload into JSON-API format.

  The optional `modelName` indicates which data of the input
  payload should be the primary data of the normalized payload.
  If no `modelName` is passed to `store.normalizePayload`, then
  this parameter is `null`.

  @param store {DS.Store}
  @param modelName {String} optional name of the primary model
  @param payload {Object}
  @return Object JSON-API document
*/
normalizePayload(store, modelName, payload)
```

## `store.pushRef`

Add a `store.pushRef` which takes a JSON-API document and returns a
`RecordReference` or `RecordArrayReference`, depending on if the primary data
is a single resource or an array:

``` js
DS.Store.reopen({

  /**
    Push given JSON-API document into the store, and get the
    reference(s) to the pushed primary data.

    Use this method if data should be pushed into the store,
    without the need for getting the pushed records. This is
    potentially faster than `store.push`, since unnecessary
    materialization of records is avoided.

    @param payload {Object} JSON-API document
    @return {DS.RecordReference|DS.RecordArrayReference} pushed primary data
  */
  pushRef(payload)

});
```

# Code Samples

The new methods can the be used to push a payload into the store and get the
relevant data:

``` js
let normalized = store.normalizePayload(payloadFromWebSocket);
let pushedRecords = store.push(normalized);
```

Or using references instead to get the associated `meta`:

``` js
// payload === {
//   book: {
//     id: 1,
//     author: "Tobias Fünke",
//     _meta: {
//       revision: "1-tmim"
//     }
//   },
//   family: {
//     id: 1,
//     name: "Bluth"
//   }
// }
let normalized = store.normalizePayload("book", payload);
let bookRef = store.pushRef(normalized); // DS.RecordReference
let book = bookRef.value(); // DS.Model
let meta = bookRef.meta();
```

And pushing array data:

``` js
// payload === {
//   people: [{
//     id: 1,
//     name: "George Michael"
//   }, {
//     id: 2,
//     name: "Buster"
//   }],
//   _meta: {
//     count: 100
//   }
// }
let normalized = store.normalizePayload("person", payload);
let peopleRef = store.pushRef(normalized); // DS.RecordArrayReference
let people = peopleRef.value(); // DS.RecordArray
let ids = peopleRef.ids();
let meta = peopleRef.meta();
```
# How We Teach This

Update the dedicated section [in the
guides](https://guides.emberjs.com/v2.7.0/models/pushing-records-into-the-store/)
to outline how data can be pushed into the `store` besides the adapter. Also,
the difference between `store.push` and `store.pushRef` and their different use
cases need to be explained.

Inform addon authors (blog post, ...) about new `normalizePayload` hook on the
serializer; with the following upgrade path:

- extract normalize functionality within `pushPayload` into `normalizePayload`
- use `normalizePayload` within `pushPayload`
- release party :tada: :balloon:

# Drawbacks

The new `serializer#normalizePayload` is another `normalize` hook which needs
to be implemented by ember data addon authors and app developers. The
difference to the `normalizeXXXResponse` hooks might not be immediately clear.
As pointed out [in
#3838](https://github.com/emberjs/data/pull/3838#issuecomment-147706227) and
[in #3676](https://github.com/emberjs/data/issues/3676), there was a
`serializer#normalizePayload` in `1.13` which has been renamed to
`normalizeResponse` instead. Re-adding a `normalizePayload` is confusing.

As mentioned already, the patch re-using `serializer.pushPayload` only works if
`store.push` is used only once. Though a check for multiple used `store.push`es
within `serializer.pushPayload` can be added, it is not 100% certain that all
use cases are supported with this.

`store.pushRef` returning `RecordReference`(s) is the first method which
introduces those type of reference to the public
([`store.getReference`](http://emberjs.com/api/data/classes/DS.Store.html#method_getReference)
is not really helpful at the moment). It is questionable if returning
references is needed here. Since we have a `store.normalizePayload` now, it is
possible to normalize a payload and get the corresponding models via
`store.push`. There is no immediate need for `store.pushRef`. But if a pushing
data into the store without materializing records should be possible, having a
`pushRef` is needed. So it basically it would boil down to this:

- use `store.push` if you want the `DS.Model`s
- use `store.pushRef` if you don't need the materialized `DS.Model`s

# Alternatives

As outlined in
[#3576](https://github.com/emberjs/data/issues/3576#issuecomment-124051628),
normalizing a payload and pushing it into the store can be done via:

``` js
let model = store.modelFor(modelName);
let serializer = store.serializerFor(modelName);
let normalized = serializer.normalizeSingleResponse(store, model, payload);
let record = store.push(normalized);
```

But this is verbose and cumbersome. Also, you need to know beforehand if the
primary data of the payload represents a single resource or if it describes a
resource collection.

# Unresolved questions

- should `store.pushPayload` return the pushed records, to be consistent with
  `store.push`?
- should `store.pushRef` return array of `RecordReference`s instead? What
  should happen when different primary types are pushed?

``` js
store.pushRef({
  data: [
    { type: "person", id: 1 },
    { type: "family", id: 2 }
  ]
});
```
- materialization mentioned in the added docs might be the first time this is
  mentioned in public (internal model might be unknown to most ember data
  users, with good reason)
