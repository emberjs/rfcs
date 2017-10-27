- Start Date: 2017-19-23
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Provide a simple way to load a relationship even if the server payload does not contain `links` or `data` for that relationship

# Motivation

When loading async relationships, Ember Data calls different adapter hooks depending on whether this are `links` or `data` in the normalized payload.

However, there is currently no adapter hook called if neither `links` or `data` are present in the normalized payload. It is possible to get Ember Data to call `findBelongsTo` or `findHasMany` by inserting `links` in the serializer, but this is not ideal.

The issue is described with a little more detail in [this Ember Data issue by @tomdale][], and in even more detail in [my blog post about how get around this][].

# Detailed design

Two major changes will need to happen in order to support this.

1. Invert the logic for deciding when to load a relationship by `link` vs by `data`.
2. Add a new adapter hook to be called when there is no `data`.

### Inverting the decision logic

Currently, Ember Data's relationship code only calls `findHasMany` or `findBelongsTo` if there is a `link` and `this.hasLoaded` is falsy.

Here are links to the [`belongs-to` `getRecord()` logic][] and the [`has-many` `getRecords()` logic][]. Note that `findHasMany` and `findBelongsTo` are called via `findLink`, which calls `fetchLink`. There is similar logic in [`belongs-to` `reload()`][] and [`has-many` `reload()`][].

This logic will need to change so that it calls `findRecord` or `findRecords` only when `hasData` is true and to otherwise call the new hook.

### New Hooks

The new hook I would like to propose be called when there is no `data`, is `findRelationship`. I have frequently written the exact same logic in `findHasMany` and `findBelongsTo`, so it would be nice to have a common hook with a default implementation that delegates to the more specific hooks.

The default implementation of `findRelationship` would then call one of two new other new hooks: `findBelongsToRelationship` and `findHasManyRelationship`. Like `findRelationship`, these would be designed to be called with or without a link and the default implementation would call the existing `findBelongsTo` and `findHasMany` when a link _is present. This would provide backwards compatibility.

NOTE: In order to find the link, it would need to be added to `relationshipMeta`, or the entire relationship object would need to be passed to `findRelationship`.

#### Default implementation of `findRelationship`, `findBelongsToRelationship`, and `findHasManyRelationship`

```js
findRelationship(store, snapshot, relationship) {
  if (relationship.kind === 'hasMany') {
    return this.findBelongsToRelationship(store, snapshot, relationship);
  } else if (relationship.kind === 'belongsTo') {
    return this.findHasManyRelationship(store, snapshot, relationship);
  }
},

findBelongsToRelationship(store, snapshot, relationship) {
  let link = getLinkFromRelationship(relationship);
  if (link) {
    return this.findBelongsTo(store, snapshot, link, relationship.relationshipMeta);
  }
},

findHasManyRelationship(store, snapshot, relationship) {
  let link = getLinkFromRelationship(relationship);
  if (link) {
    return this.findHasMany(store, snapshot, link, relationship.relationshipMeta);
  }
},
```

I think this is basically the solution proposed by @tomdale in [emberjs/data#2162][].

# How We Teach This

TBD

# Drawbacks

* `findBelongsToRelationship` and `findHasManyRelationship` are pretty long
  names.

  It would be nice to call them `findBelongsTo` and `findHasMany`, but we need to keep them for backwards compatability. We could check the arity of existing implementations of `findBelongsTo` and `findHasMany`, but I would worry that they have been implemented with exactly three arguments `(store, snapshot, link)`, which would be impossible to distingish from `(store, snapshot, relationship)`.

# Alternatives

* The same approach could be written without adding so many hooks `findBelongsToRelationship` and `findHasManyRelationship`.

  ```js
  findRelationship(store, snapshot, relationship) {
    let link = getLinkFromRelationship(relationship);
    if (link) {
      if (relationship.kind === 'hasMany') {
        return this.findBelongsTo(store, snapshot, link, relationship.relationshipMeta);
      } else if (relationship.kind === 'belongsTo') {
        return this.findHasMany(store, snapshot, link, relationship.relationshipMeta);
      }
    }
  },
  ```

  This would limit the size of the API and still provide the necessary hook, but requires a bit more boilerplate if there needs to be different logic between `hasMany` and `belongsTo`.

# Unresolved questions

* What should we call these hooks? I would actually love to call them `findHasMany` and `findBelongsTo`, but these hooks already exist and it would not be backwards compatible to change them.
* How would `reload()` work?

[this Ember Data issue by @tomdale]: https://github.com/emberjs/data/issues/2162
[my blog post about how get around this]: http://www.amielmartin.com/blog/2017/07/17/how-ember-data-loads-async-relationships-part-3/

[`belongs-to` `getRecord()` logic]: https://github.com/emberjs/data/blob/v2.16.0/addon/-private/system/relationships/state/belongs-to.js#L151-L156
[`has-many` `getRecords()` logic]: https://github.com/emberjs/data/blob/v2.16.0/addon/-private/system/relationships/state/has-many.js#L259-L264

[`belongs-to` `reload()`]: https://github.com/emberjs/data/blob/v2.16.0/addon/-private/system/relationships/state/belongs-to.js#L178-L187
[`has-many` `reload()`]: https://github.com/emberjs/data/blob/v2.16.0/addon/-private/system/relationships/state/has-many.js#L173-L177

[emberjs/data#2162]: https://github.com/emberjs/data/issues/2162

