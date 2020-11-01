---
Start Date: 2015-05-20
RFC PR: https://github.com/emberjs/rfcs/pull/57
Ember Issue: https://github.com/emberjs/data/pull/3303

---

# Summary

This RFC adds a unified way to perform meta-operations on records, has-many relationships and belongs-to relationships:

* get the current local data synchronously without triggering a fetch or producing a promise
* notify the store that a fetch for a given record has begun, and provide a promise for its result
* similarly, notify a record that a fetch for a given relationship has begun, and provide a promise for its result
* retrieve server-provided metadata about a record or relationship

# Motivation

When we initially designed the Ember Data API for relationships, we focused on consumption and mutation of the relationship data. For example, you can retrieve the value of a `belongsTo` relationship via `get('post')`, or adding new records to a has-many relationship via `get('comments').pushObject(newComment)`.

The top-level reading operations are designed to be [zalgo][zalgo]-proof: regardless of whether or not the record or relationship has been loaded already, you get back a promise for the result. Behind the scenes, this will trigger a fetch if needed, or simply return the current value if it has already been fetched. From a programming model perspective, this simplifies your code because you can handle locally-available data and remotely-available data in a single code path.

[zalgo]: http://blog.izs.me/post/59142742143/designing-apis-for-asynchrony

However, in sophisticated applications, there is often a need to refer to a record without triggering side effects.

For example, you may want to initiate the fetch for a record or relationship yourself, and provide Ember Data with a promise representing the result of that fetch. That use-case is supported by the `store.push` API, but it has a few problems:

* The `store.push` API supports pushing data once the fetch has completed, but no way of telling
  Ember Data that a fetch has begun. As a result, any calls to `store.find` in the interim will
  trigger unnecessary fetches.
* The `store.push` API works only for top-level records with already-known types and IDs. It does
  not support any way of "feeding" the data for a relationship to Ember Data.

In sum, this makes it difficult to front-load work (especially asynchronous work). Instead, Ember Data is currently optimized for reacting to requests from the application layer, which is sometimes a very awkward way of structuring your code.

Second, Ember Data was originally designed with APIs that refer to data and operations on data. Over time, we have come to realize that people quite often need to look at metadata about records or relationships, as well as perform meta-operations on them.

Some examples:

* getting the expected count of a has-many relationship before it has been fetched
* learning whether a relationship is already loaded or not
* examining server-sent metadata
* working with pages of records in a has-many relationship, especially when pages are loaded asynchronously ("pagination")

Third, because has-many relationships are represented as a `RecordArray`, we have been able to kludge around some of these issues by adding meta-operations to the has-many relationship itself. In contrast, belongs-to relationships have remained anemic. For example, there is no way to trigger a reload of a belongs-to relationship, whereas has-many relationships can be reloaded by calling `.reload()` on the `RecordArray`.

# Detailed design

This RFC proposes the addition of three new **public** APIs:

* `RecordReference`
* `HasManyReference`
* `BelongsToReference`

## Getting References

* `store.getReference(type, id)`
* `record.getReference(name)`

## References

### `push`

```js
/**
  This API allows you to provide a reference with new data. The simplest usage
  of this API is similar to `store.push`: you provide a normalized hash of data
  and the object represented by the reference will update.

  If you pass a promise to `push`, Ember Data will not ask the adapter for the
  data if another attempt to fetch it is made in the interim. When the promise
  resolves, the underlying object is updated with the new data, and the promise
  returned by *this function* is resolved with that object.

  For example, `recordReference.push(promise)` will be resolved with a record.

  @method
  @param {Promise|Object}
  @returns Promise<T> a promise for the value (record or relationship)
*/
```

### `pushPayload`

```js
/**
  This API is similar to `push`, but it invokes the serializer with the
  resolved data. This makes it possible to share normalization logic
  across multiple calls to `pushPayload` or between proactive pushes
  and reactive responses from the adapter.

  @method
  @param {Promise|Object}
  @returns Promise<T> a promise for the value (record or relationship)
*/
```

### `state`

```js
/**
  The current state of the entity, based on the semantics of the
  entity in question. For records, this should expose a subset of
  the named states in the internal state machine.

  @property
  @type String
*/
```

### `value`

```js
/**
  If the entity referred to by the reference is already loaded, it
  is present as `reference.value`. Otherwise, the value of this
  property is `null`.

  @property
*/
```

### `data`

```js
/**
  The value of the (normalized) representation of this entity. For
  example, `recordReference.data` will return a normalized dictionary
  of attributes and links.

  @property
*/
```

### `metadata`

```js
/**
  The most recent value of the metadata returned by the server for
  the value represented by this reference.

  @property
*/
```

### `load`

```js
/**
  Triggers a fetch for the backing entity based on its `remoteType`
  (see `remoteType` definitions per reference type).

  @method
  @param {Object} an options hash, similar to the one currently
    passed to `store.find`.
*/
```

### `unload`

```js
/**
  Unload the entity referred to by this relationship. After this
  operation, its `value`, `data` and `metadata` members will return
  to `null`, and the record itself will be purged from the identity
  map.

  @method
*/
```

## RecordReference

### `remoteType`

```js
/**
  How the reference will be looked up with it is loaded:

  * `link`, a URL
  * `identity`, by the `type` and `id`
*/
```

### `type`

```js
/**
  The type of the record that this reference refers to.

  @property
*/
```

### `id`

```js
/**
  The `id` of the record that this reference refers to.

  Together, the `type` and `id` properties form a composite key
  for the identity map.

  @property
*/
```

## HasManyReference

### `remoteType`

```js
/**
  How the reference will be looked up when it is fetched:

  * `link`, a URL provided by the server
  * `ids`, a list of IDs provided by the server
  * `dynamic`, a dynamic URL will be created based on the identity
    of the parent.

  @property
*/
```

### `link`

```js
/**
  If the `remoteType` is `link`, the URL to use to load the relationship.

  @property
*/
```

### `ids`

```js
/**
  If the `remoteType` is `ids`, a list of IDs that is used to formulate
  the query to the server (together with `type`).

  @property
*/
```

### `type`

```
/**
  The model type represented by this relationship.

  @property
*/
```

### `parent`

```js
/**
  A reference to the record that has this `hasMany` on it.

  @property
*/
```

### `inverse`

```js
/**
  If there is an inverse relationship, this property is a reference
  to it.

  @property
*/
```

## BelongsToReference

### `remoteType`

```js
/**
  How the reference will be looked up when it is fetched:

  * `link`, a URL provided by the server
  * `id`, an ID to use to form the URL
  * `dynamic`, a dynamic URL will be created based on the identity
    of the parent.

  @property
*/
```

### `link`

```js
/**
  If the `remoteType` is `link`, the URL to use to load the relationship.

  @property
*/
```

### `ids`

```js
/**
  If the `remoteType` is `id`, an ID that is used to formulate
  the query to the server (together with `type`).

  @property
*/
```

### `type`

```
/**
  The model type represented by this relationship.

  @property
*/
```

### `parent`

```js
/**
  A reference to the record that has this `belongsTo` on it.

  @property
*/
```

### `inverse`

```js
/**
  If there is an inverse relationship, this property is a reference
  to it.

  @property
*/
```


# Drawbacks

The main drawback to this proposal is that it adds significant surface area to Ember Data, which could easily be perceived as significant additional complexity. However, we believe that the unification of the various entities in Ember Data, as well as exposing internal tools that were previously only available to the store, will actually reduce the complexity of many common patterns.

# Alternatives

The main alternative is to address each use case with a new API:

* `store.peek(record, id)`, `record.peek(relationship)` to retrieve the current value of the relationship only if it was loaded
* extend `store.push` and `store.pushPayload` to take promises
* APIs like `record.inverseFor(relationship)`, `record.typeFor(relationship)`, etc.
* APIs like `record.idsFor(relationship)`, `record.metadataFor(relationship)`, and `store.metadataFor(type, id)`

We believe that the cumulative overhead of all of these APIs is far more than the overhead of the reference APIs.

# Unresolved questions

Is there a need to represent "prefetch metadata" separately? This is metadata that the app knows about before fetch, and which it would want to persist through an `unload()` operation (along with identity information like type, id and link).
