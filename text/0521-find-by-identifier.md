---
Start Date: (fill me in with today's date, 2019-08-29)
Relevant Team(s): EmberData
RFC PR: https://github.com/emberjs/rfcs/pull/521
Tracking: (leave this empty)

---

# [DATA] findRecord/peekRecord via Identifier

## Summary

Users should be able to peek or find records based on `lid`.

## Motivation

Apps and Addons making use of [Identifiers](https://github.com/emberjs/rfcs/pull/403)
may wish to peek or find the record for a known `identifier`. This
RFC would allow them to do so.

## Detailed design

A new method signature would be added to `findRecord`
`peekRecord` and `getReference`. The existing signatures would not be
deprecated at this time.

For the case where calling `findRecord` would result in a request
being necessary but either no `type` or `id` information being known
we would error.

This is because there is no meaningful way for the store to create a
network request for a record that does not exist yet (a
`NewResourceIdentifierObject`), or for one for which the only thing
we know is a locally assigned id (`Identifier`).

```ts
interface Identifier {
  lid: string;
}

export interface ExistingResourceIdentifierObject {
  id: string;
  type: string;
  lid?: string;
}

export interface NewResourceIdentifierObject {
  id: string | null;
  type: string;
  lid: string;
}

export type Peekable =
  | Identifier
  | ExistingResourceIdentifierObject
  | NewResourceIdentifierObject;

class Store {
  peekRecord(type: string, id: string | number): Record | null {}
  peekRecord(identifier: Peekable): Record | null {}

  findRecord(type: string, id: string | number, options?: any): PromiseRecord {}
  findRecord(identifier: Peekable, options?: any): PromiseRecord {}

  getReference(modelName: string, id: string | number): RecordReference {}
  getReference(identifier: Peekable): RecordReference {}
}
```

## How we teach this

For existing usage of these methods no migration or changes are necessary. For folks wishing to use the new
APIs an example route is below that both peeks a record and makes a findRecord request.

**current style**

```ts
{
  @service('store') store;

  model({ user_id, post_id }) {
    let user = this.store.peekRecord('user', user_id);

    return hash({
        user,
        post: this.store.findRecord('post', post_id)
    });
  }
}
```

**using RecordIdentifiers instead**

```ts
{
  @service('store') store;

  model({ user_id, post_id }) {
    let user = this.store.peekRecord({ type: 'user', id: user_id });

    return hash({
        user,
        post: this.store.findRecord({ type: 'post', id: post_id })
    });
  }
}
```

If an `lid` is known the `RecordIdentifier` passed to these methods can provide it
in addition to or in place of `id`.

## Drawbacks

- It requires the user to create an object as an argument when seeking to use identifiers.
  However if the user does not do so we still do ourselves almost immediately, here we are
  shaping the object sooner.

## Alternatives

- `peekRecord(null, null, lid)` rejected for clumsy ergonomics
- `findRecord(identifier)` attempting to request with just an lid,
  rejected until reconsideration at such time as our request fulfillment doesn't rely heavily on per-type adapters
- `peekRecord(identifier)` but no `findRecord(identifier)` rejected because of the utility in these APIs mirroring each other, especially for relationships.
- `store.recordForIdentifier(identifier)` instead of these changes, rejected because we already have a `peek` API.
