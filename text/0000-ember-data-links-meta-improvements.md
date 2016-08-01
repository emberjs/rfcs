- Start Date: 2016-07-29
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Ember Data uses JSON-API internally: this RFC aims to provide a public API for
accessing and interacting with the already available data.

Meta and links are made available via references for record, belongs-to,
has-many and record-arrays (using the new `RecordArrayReference`). All those
references will expose their corresponding meta and links via – surprise –
`meta()` and `links()` methods. Since meta is only a plain JavaScript object,
there is no need for further abstraction. A link on the other hand is
represented as a `LinkReference`, which allows to get the related meta and to
load the link via `load()`.

Since references are not observable by design, hooks are proposed in the
accompanying [RFC #123](https://github.com/emberjs/rfcs/pull/123) which allow
to retrieve data from the reference, derive some properties and set it on the
relevant model instances.

The references API is intentionally designed to be a low level API. It can be
used in a higher level API's provided by add-ons; or even in ember-data itself,
once a use case is flushed out and it's reasonable to being in core.

#### In short

- Add `LinkReference` which is an abstraction for a single link and its
  properties (meta, name, href) and action (load)
- Add a `RecordArrayReference` which abstracts a collection of records for its
  properties (meta, links) and action (reload)
- A new `links()` method is added to record, belongs-to, has-many and
  record-array references which returns all associated links
- Hooks which allow to react to changes when the backing data of references
  changes (hooks might not be part of this RFC, but are already somewhat
  addressed in [RFC#123](https://github.com/emberjs/rfcs/pull/123))

# Motivation

## Status quo

Currently meta is only available on relationships via the `belongs-to` and
`has-many` references. Record level meta data is not available as `meta` on the
`DS.Model` instance, because the source for record level meta data can come
from various places: `store.queryRecord`, `store.findRecord` or even via
`store.push` and `store.pushPayload`. A problem with setting `meta`
automatically as property on the `DS.Model` instance might overwrite previously
set meta of a different call.

Meta is available on the `ManyArray` of a has many relationship. It is also set
on the `AdapterPopulatedRecordArray`s of `store.query`. It is however currently
not possible to get the `meta` for a `store.findAll`.

In terms of links currently only `related` is handled on relationships. Though
the [`ds-links-in-record-array`
feature](https://github.com/emberjs/data/pull/4263) makes links available on
the `AdapterPopulatedRecordArray` of a `store.query`, Ember Data doesn't offer
a dedicated API to load the `next` link for example to support pagination.
Also, according to JSON-API specification, links can have `meta` too, which is
also not supported in core Ember Data.

## Proposal

- allow to interact with links for single resource and resource collections
- make references consistent, so they are available for record arrays and links
- provide low level API for internally used JSON-API
- provide hooks to react to changes in references
  ([RFC#123](https://github.com/emberjs/rfcs/pull/123))

# Detailed design

### `DS.LinkReference`

This new reference is added and represents the reference to a single link entry
of the JSON-API `links: {}` object:

```js
/**
  Reference to a single link of a JSON-API links object.
*/
class LinkReference {

  /**
    Get the name of the link.

    E.g. "related", "self" or "next"

    @return {String} name of the link
  */
  name()

  /**
    Get the href of the link.

    @return {String} href
  */
  href()

  /**
    Get the meta associated with the link.

    @return {Object} meta
  */
  meta()

  /**
    Load the link.

    Returned promise resolves with a `DS.RecordArray`.

    @return {Promise}
  */
  load()

  /**
    Get the reference to which this link is connected to.

    This can either be a `RecordReference`, `BelongsToReference`,
    `HasManyReference` or `RecordArrayReference`.

    @return {DS.Reference} reference to which this link
                           is connected to
  */
  parentRef()

}
```

### `DS.RecordArrayReference`

This new type references an array of records, returned for example by
`store.query` or `store.findAll`.

```js
class RecordArrayReference {

  /**
    Get the `DS.RecordArray` which holds all the `DS.Model`s.

    This can either be the result for a `store.query` or a
    `store.findAll`.

    @return {DS.RecordArray}
  */
  value()

  /**
    Reload the underlying `RecordArray`.

    This will re-fetch the `self` link.

    @return {Promsise} resolving with the reloaded `DS.RecordArray`
  */
  reload()

  /**
    Get all links associated with this `DS.RecordArray`.

    This is useful for pagination for example.

    @return {Array<DS.LinkReference>}
  */
  links()

  /**
    Get the meta associated with the underlying `DS.RecordArray`

    @return {Object}
  */
  meta()

}
```

### `RecordReference.meta()`

The `RecordReference` gets a new `meta()` method, which returns the last
associated meta for the record pushed into the store via `findRecord()`,
`queryRecord()`, `store.push()` or `store.pushPayload()` and `createRecord()`
and `updateRecord()`.

```js
// GET /books/1
// {
//   data: { … },
//   meta: {
//     isThriller: true
//   }
// }
store.findRecord("book", 1).then(function(book) {
  // get the RecordReference
  let bookRef = book.ref();

  let meta = bookRef.meta();
  assert.equal(meta.isThriller, true);
});
```

### `[Record|BelongsTo|HasMany|RecordArray]Reference.links()`

The `links()` method on those references allows to get all link references
associated with the reference, or the specific `LinkReference`, if the name of
the link is passed:

```js
// {
//   data: {
//     type: "book",
//     id: 1,
//     relationships: {
//       chapters: {
//         data: [ … ],
//         links: {
//           "next": { … },
//           "prev": { … }
//         }
//       }
//     }
//   }
// }
let chaptersRef = store.peekRecord("book", 1).hasMany("chapters");

// [<DS.LinkReference>, <DS.LinkReference>]
let links = chaptersRef.links();

// DS.LinkReference
let nextLink = chaptersRef.links("next");
```

### Adaptions to current references / models

To fully complete the circle of references, the existing classes are modified
as follows:

- `RecordReference`
  - add `links()`
- `Model`
  - add `ref()` to get the corresponding `RecordReference`
- `BelongsToReference`
  - add `links()`
  - add `parentRef()` to get a reference to the parent `RecordReference`
- `HasManyReference`
  - add `links()`
  - add `parentRef()` to get a reference to the parent `RecordReference`
- `ManyArray`
  - add `ref()` to get the corresponding `HasManyReference`
- `AdapterPopulatedRecordArray` and `RecordArray`
  - add `ref()` to get the corresponding `RecordArrayReference`

### Hooks

Additionally to the new types of references and the extension of the existing
ones, the Model Lifecycle Hooks proposed in
[RFC#123](https://github.com/emberjs/rfcs/pull/123) allow to react to changes
for the underlying data of references. The hooks are explained in greater
detail in that RFC, but record level meta data handling for example might look
like this:

```js
// app/models/books.js
import Model from "ember-data/model";

const Book = Model.extend({});

Book.reopenClass({

  didReceiveData(recordRef) {
    let book = recordRef.value();
    let meta = recordRef.meta();

    if (book && meta) {
      // set meta on the DS.Model instance so it can be
      // accessed in the template
      book.set("meta", meta);
    }
  }

});

export default Book;
```

## Sample code

### Pagination

```js
let allPages = Ember.A();

// GET /books?page=1
store.query("book", { page: 1 }).then(function(books) {
  allPages.addObjects(books.toArray());

  let booksRef = books.ref();
  let nextPage = booksRef.links("next");

  // GET /books?page=2
  nextPage.load().then(function(nextBooks) {
    allPages.addObjects(nextBooks.toArray());
  });
});
```

### Meta for `findAll`

```js
// GET /books
store.findAll("book").then(function(books) {
  // DS.RecordArrayReference
  let booksRef = books.ref();

  let meta = booksRef.meta();
});
```

# How we teach this

- for the beginning not in the guides, until this is fleshed out and seems
  stable / usable
- in the beginning only API docs within the source

# Drawbacks

- massive increase of low level, public API
- though it extends the concept of references, it's a significant increase of
  API surface

# Alternatives

- N/A

# Unresolved questions

- is `DS.LinkReference` actually a `DS.Link` and not really a reference?
- rename `name()` of `LinkReference`? What is the official terminology for it?
- what type of `RecordArray` does `LinkReference#load()` resolve with? Do we
  need a new type or is `DS.AdapterPopulatedRecordArray` sufficient?
- should `books.hasMany("chapters").links("next").load()` update the content of
  the relationship? Should there be a dedicated API for this, e.g.
  `books.hasMany("chapters").loadLink("next")`?
