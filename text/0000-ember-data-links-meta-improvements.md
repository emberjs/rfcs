- Start Date: 2016-07-29
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Ember Data uses JSON-API internally: this RFC aims to provide a public API for
accessing and interacting with the already available data.

Meta and links are made available via references for record, belongs-to,
has-many and JSON-API documents (using the new `DocumentReference`). All those
references will expose their corresponding meta and links via – surprise –
`meta()` and `links()` methods. Since meta and links are only a plain
JavaScript objects, there is no need for further abstraction.

Since references are not observable by design, hooks are proposed in the
accompanying [RFC #123](https://github.com/emberjs/rfcs/pull/123) which allow
to retrieve data from the reference, derive some properties and set it on the
relevant model instances.

The references API is intentionally designed to be a low level API. It can be
used in a higher level API's provided by add-ons; or even in ember-data itself,
once a use case is flushed out and it's reasonable to being in core.

#### In short

- A new `links()` method is added to record, belongs-to, has-many and
  document references which returns all associated links
- Add `LinkObject` which is an abstraction for a single link and its
  properties (meta, href)
- Add a `DocumentReference` which describes a JSON-API document and its
  properties (meta, links, data)
- Hooks which allow to react to changes when the backing data of references
  changes (hooks might not be part of this RFC, but are already somewhat
  addressed in [RFC#123](https://github.com/emberjs/rfcs/pull/123))

# Motivation

## Status quo

### Links

In terms of links currently only `related` is handled on relationships. Though
the [`ds-links-in-record-array`
feature](https://github.com/emberjs/data/pull/4263) makes links available on
the `AdapterPopulatedRecordArray` of a `store.query`, Ember Data doesn't offer
a dedicated API to load the `next` link for example to support pagination.
Also, according to JSON-API specification, links can have `meta` too, which is
also not supported in core Ember Data.

### Meta

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

The JSON-API specification states that meta information can be present in
various locations on a JSON-API document.

#### Single resource

```js
{
  data: {
    id: 1,
    type: "book",
    relationships: {
      author: {
        data: { … },

        // relationship level meta
        meta: {
          lastUpdatedAt: "2016-08-13T12:34:56Z"
        }
      }
    },

    // resource level meta
    meta: {
      lastUpdatedAt: "2016-08-13T12:34:56Z"
    },

    links: {
      self: {
        href: "…",

        // link level meta
        meta: {
          canBeCached: false
        }
      }
    }
  },

  // document level meta
  meta: {
    apiRateLimitRemaining: 35
  }
}
```

#### Resource collection

```js
{
  data: [{
    id: 1,
    type: "book",
    relationships: {
      author: {
        data: { … },

        // relationship level meta
        meta: {
          lastUpdatedAt: "2016-08-13T12:34:56Z"
        }
      }
    },

    // resource level meta
    meta: {
      lastUpdatedAt: "2016-08-13T12:34:56Z"
    },

    links: {
      self: {
        href: "…",

        // link level meta
        meta: {
          canBeCached: false
        }
      }
    }
  }],

  // document level meta
  meta: {
    total: 123,
    apiRateLimitRemaining: 34
  }
}
```

While a resource collection has its meta on the document level, a single
resource might have resource specific meta data under `data.meta`, where the
response specific meta data is located in the root level `meta`. Both meta data
need to be accesible for a record.

## Proposal

- allow to interact with links for single resource and resource collections
- make references consistent, so they are available for links and documents
- provide low level API for internally used JSON-API
- provide hooks to react to changes in references
  ([RFC#123](https://github.com/emberjs/rfcs/pull/123))

# Detailed design

## Links

Links are exposed in an object, with each link exposed with the following
properties:

```js
/**
  Link
*/
{

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

}
```

### `[Record|BelongsTo|HasMany|Document]Reference.links()`

The `links` method on those references allows us to get the links object.

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

// Object containing links
let links = chaptersRef.links();

// LinkObject
let nextLink = chaptersRef.links().next;
```

## Meta

### `DS.DocumentReference`

This new type represents a JSON-API document:

```js
class DocumentReference {

  /**
    Get all document level links.

    This is useful for pagination of resource collections for example.

    @return {Array<DS.LinkReference>}
  */
  links()

  /**
    Get the document level meta.

    @return {Object}
  */
  meta()

  /**
    Get the references, associated with the data for the document.

    @return {DS.RecordReference|Array<DS.RecordReference>}
  */
  data()

}
```

### `RecordReference.meta()`

The `RecordReference` gets a new `meta()` method, which returns the *last
associated record level meta* for the record pushed into the store:

```js
// GET /books/1
// {
//   data: {
//     id: 1,
//     type: "book",
//     attributes: { … },
//     relationships: { … },
//     meta: {
//       lastUpdatedAt: "2016-08-13T12:34:56Z"
//     }
//   }
// }
store.findRecord("book", 1).then(function(book) {
  // get the RecordReference
  let bookRef = book.ref();

  let meta = bookRef.meta();
  assert.ok(meta.lastUpdatedAt);
});

// GET /books
// {
//   data: [{
//     id: 1,
//     type: "book",
//     meta: {
//       isPublished: true
//     }
//   }]
// }
store.query("book", {}).then(function(books) {
  // get the RecordReference of first book in result
  let bookRef = books.objectAt(0).ref();

  let meta = bookRef.meta();
  assert.ok(meta.isPublished);
});
```

## Adaptions to current references / models

To fully complete the circle of references, the existing classes are modified
as follows:

- `RecordReference`
  - add `meta()`
  - add `links()`
  - add `documentRef()`
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
  - add `documentRef()`

## Hooks

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

### Meta for `findAll`

```js
// GET /books
// {
//   data: […],
//   meta: {
//     total: 123
//   }
// }
store.findAll("book").then(function(books) {
  // DS.DocumentReference
  let booksRef = books.documentRef();

  let meta = booksRef.meta();
  assert.ok(meta.total);
});
```

### Meta for record (record and document level)

```js
// GET /books/1
// {
//   data: {
//     id: 1,
//     type: "book",
//     meta: {
//       lastUpdatedAt: "2016-08-13T12:34:56Z"
//     }
//   },
//   meta: {
//     apiRateLimitRemaining: 33
//   }
// }
store.findRecord("book", 1).then(function(book) {
  let bookRef = book.ref();
  let bookMeta = bookRef.meta();
  assert.ok(bookMeta.lastUpdatedAt);

  let docRef = bookRef.documentRef();
  let docMeta = docRef.meta();
  assert.ok(docMeta.apiRateLimitRemaining > 0);
});
```

# How we teach this

- for the beginning not in the guides, until this is fleshed out and seems
  stable / usable
- in the beginning only API docs within the source

# Drawbacks

- increase of low level, public API
- though it extends the concept of references, it's a significant increase of
  API surface

# Alternatives

- N/A

# Unresolved questions

- what is the API to load a link?
- since multiple "locations" of meta will be supported, there needs to be a new
  hook only within the `RESTSerializer` to extract record/document level meta,
  additionally to the `extractMeta` hook, which is invoked with the whole
  payload
