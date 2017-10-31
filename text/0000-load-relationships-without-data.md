- Start Date: 2017-19-23
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Provide a simple way to load a relationship even if the server payload does not contain `links` or `data` for that relationship.

# Motivation

### Background

When loading async relationships, Ember Data calls different adapter hooks depending on whether these are `links` or `data` in the normalized payload.

However, there is currently no adapter hook called if neither `links` or `data` are present in the normalized payload. It is possible to get Ember Data to call `findBelongsTo` or `findHasMany` by inserting `links` in the serializer, but this is not ideal.

The issue is described with a little more detail in [this Ember Data issue by @tomdale][], and in even more detail in [my blog post about how get around this][].

### Why would you want/need to load relationships when there are no `links` or `data`?

There are a few scenarios that I have run in to. Usually the base issue is that not everyone has control over the API.

1. Sometimes the server does not provide `links` and you want to define the relationship url in the adapter

   [ember-data-url-templates has a new feature][] that makes this simple, and I think makes a good example:

   ```js
   // app/models/post.js
   reactions: hasMany('reactions', { urlTemplate: 'reactions' }),

   // app/adapters/post.js
   reactionsUrlTemplate: "/posts/{id}/reactions",
   ```

2. > where the relationship should be populated by fetching a URL, but the URL is derived from information contained in the record, rather than being provided in the payload, hypermedia-style. - @tomdale

   This is basically true for the previous example, but the only state derived from the record is the id.

   In an application I work on, we have a model that represents a report query. The same query parameters can be used to query many different reports. Each report is a relationship on the query model. We use the appropriate attributes on the query model in the query params for each relationship url.

3. Building on the previous example, sometimes you want to load relationships for brand new records that haven't been persisted to the server

   In order to get the previous example with the query model to work, the query is saved to the server so that the response gets normalized through the serializer, which can then inject `relationships.links.related`.

# Detailed design

Two major changes will need to happen in order to support this.

1. Invert the logic for deciding when to load a relationship by `link` vs by `data`
2. Add a new adapter hook to be called when there is no `data`

### Inverting the decision logic

Currently, Ember Data's relationship code only calls `findHasMany` or `findBelongsTo` if there is a `link` and `this.hasLoaded` is falsey.

Here are links to the [`belongs-to` `getRecord()` logic][] and the [`has-many` `getRecords()` logic][]. Note that `findHasMany` and `findBelongsTo` are called via `findLink`, which calls `fetchLink`. There is similar logic in [`belongs-to` `reload()`][] and [`has-many` `reload()`][].

This logic will need to change so that it calls `findRecord` or `findRecords` only when `hasData` is true and to otherwise call the new hook.

### New Hooks

When there is no `data`, I'd like to propose that a new `findRelationship` hook be called. I have frequently written the exact same logic in `findHasMany` and `findBelongsTo`, so it would be nice to have a common hook with a default implementation that delegates to the more specific hooks.

The default implementation of `findRelationship` would then call one of two new other new hooks: `findBelongsToRelationship` and `findHasManyRelationship`. Like `findRelationship`, these would be designed to be called with or without a link and the default implementation would call the existing `findBelongsTo` and `findHasMany` when a link _is_ present. This would provide backwards compatibility.

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

# How we teach this

This change should be completely backwards compatible, so there wouldn't need to be any modifications to existing applications. The new adapter hooks should be documented.

One thing to note is that some people are already confused about the difference between `findHasMany` and `findMany`, and so adding a `findHasManyRelationship` could add to this confusion. I think that having a specific section somewhere in documentation about the difference between these hooks might be a good idea. I'm not sure if this would belong in the guides or in the adapter API documentation, but it could be something like [my blog post](http://www.amielmartin.com/blog/2017/05/05/how-ember-data-loads-relationships-part-1/), but shorter.

# Drawbacks

`findBelongsToRelationship` and `findHasManyRelationship` are pretty long
names.

It would be nice to call them `findBelongsTo` and `findHasMany`, but we need to keep them for backwards compatability. We could check the arity of existing implementations of `findBelongsTo` and `findHasMany`, but I would worry that they have been implemented with exactly three arguments `(store, snapshot, link)`, which would be impossible to distingish from `(store, snapshot, relationship)`.

# Alternatives

The same approach could be written without adding the hooks `findBelongsToRelationship` and `findHasManyRelationship`.

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

While it requires a bit more boilerplate to add different logic between `hasMany` or `belongsTo` relationships, this would limit the expansion of the current API.

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

[ember-data-url-templates has a new feature]: https://github.com/amiel/ember-data-url-templates/pull/36
