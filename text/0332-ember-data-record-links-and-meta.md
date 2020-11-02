---
Start Date: 2018-10-24
Relevant Team(s): Ember Data
RFC PR: https://github.com/emberjs/rfcs/pull/332
Tracking: https://github.com/emberjs/rfc-tracking/issues/19

---

# Ember Data Record Links & Meta

## Summary

Enable users to associate `links` and `meta` information with individual records
  in a manner accessible via the template.

## Motivation

Sometimes users have meta or links information to associate with a specific record.
Users of the `json-api` specification will commonly understand this information as
belonging to an individual `resource`.  

While `ember-data` allows for this information to exist on relationships, it does
 not allow for it to exist on records, which has to this point been a glaring omission
 for users of `json-api` and similar specifications.

## Detailed design

In keeping with the current design of the `store.push` API which expects the `json-api` format,
users would include optional `meta` and `links` information as member properties of a resource.

```js
store.push({
  data: {
    type: 'contributor',
    id: '1',
    attributes: {},
    relationships: {},
    meta: {
      // ... <any>
    },
    links: {
      self: './person/1', // ... <String>
    }
  }
});
```

--------------------------------

`links` and `meta` will be accepted anywhere a `resource` may be encountered in a payload.

```js
store.push({
  data: [
    {
      type: 'contributor',
      id: '1',
      attributes: {},
      relationships: {
        projects: {
          data: [
            { type: 'project', id: '1' }
          ]
        }
      },
      meta: {
        // ... <any>
      },
      links: {
        self: './person/1', // ... <String>
      }
    }
  ],
  included: [
    {
      type: 'project',
      id: '1',
      attributes: {},
      relationships: {
        contributors: {
          data: [
            { type: 'contributor', id: '1' }
          ]
        }
      },
      meta: {
        // ... <any>
      },
      links: {
        self: './github-projects/1', // ... <String>
      }
    }
  ]
})
```

--------------------------------

Links & Meta on objects used as `ResourceIdentifiers` (e.g. to link to another resource within a relationship)
 will not be used for the associated resource and will be silently ignored.

```js
let record = store.push({
  data: {
    type: 'contributor',
    id: '1',
    attributes: {},
    relationships: {
      projects: {
        data: [
          {
            type: 'project',
            id: '1',         
            meta: {}, // ignored
            links: {} // ignored
          }
        ]
      }
    },
  }
});
```
--------------------------------

Links & Meta on objects provided for `Relationships` will continue to work (as they do today).

```js
let record = store.push({
  data: {
    type: 'contributor',
    id: '1',
    attributes: {},
    relationships: {
      projects: {
        data: [
          {
            type: 'project',
            id: '1',         
          }
        ],
        meta: {}, // available on the Record's hasMany relationship
        links: {} // available on the Record's hasMany relationship
      }
    },
  }
});
```
--------------------------------

`links` and `meta` properties will be exposed as getters on instances of `DS.Model` and will default to `null` if
no `meta` or `links` have been provided.

```js
let record = store.push({
  data: {
    type: 'person',
    id: '1',
    attributes: { name: '@runspired' },
    meta: {
      expiresDate: '2018-05-10'
    },
    links: {
      self: './people/runspired'
    }
  }
});

record.meta.expiresDate; // '2018-05-10'
record.links.self; // './people/runspired'
```

--------------------------------

`links` and `meta` will similarly be exposed as on instances of `Snapshot` given to
adapter and serializer methods. In keeping with `Snapshot#attributes()`, they will
be exposed as methods.  Should users desire to reload a record via link, they could
achieve such by utilizing the `links()` method to check for a link when making a request.

```js
class Snapshot {
  links() {}
  meta() {}
}
```

--------------------------------

#### The shared namespace problem and interop with existing workaround for `links` and `meta`.

The `json-api` spec places `type`, `id`, and all members of `attributes` and `relationships` into
a single shared flattened namespace.  This flattened namespace is what `records` expose.

The spec does not put `links` and `meta` into this namespace, and it is valid to have `links` and `meta`
as member names of either `attributes` or `relationships`.

Some apps have taken advantage of this to move `links` and `meta` into `attributes` on their serializer
and to expose them via `DS.attr` on their records.

The `getter` we are proposing adding to `DS.Model` would be overwriteable. In the case that there is a
 conflict, the version defined by the end user model would win. It would be up to consuming apps to
 decide whether they wish to avoid this conflict by renaming the non-resource `links` and `meta` either
 in their serializer or in their API responses.

## How we teach this

Documentation for `DS.Model` should be updated to reflect these properties, the potential conflict
 (and the default conflict resolution) explained in said documentation, and guides on working with
 Models should reflect this capability.

## Drawbacks

Users may sometimes encounter confusion when `links` or `meta` is a member of attributes or
relationships.

## Alternatives

- Rename `links` and `meta` to a name less likely to collide and which we fully reserve, such as
`recordLinks` and `recordMeta`. We felt this would be confusing.

- Enforce accessing `links` and `meta` via some other object such as the `Reference` API. In addition
  to being cumbersome and confusing, this would lack discoverability and be unergonomic in templates.

- Enforce accessing `links` and `meta` via some imported helper, e.g. `recordMetaFor(record)` or `recordLinksFor(record)`.
  We felt this would be confusing and unergonomic for templates.

## Unresolved questions

None
