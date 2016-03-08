- Start Date: 2016-03-04
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

<!--
One paragraph explanation of the feature.
-->

The basic idea of model lifecycle hooks is similar to component lifecycle
hooks. These hooks get invoked when the model's backing data changes, giving
the model a chance to compute derived properties and to copy properties from
the payload.

# Motivation

<!--
Why are we doing this? What use cases does it support? What is the expected outcome?
-->

A first implementation of [RFC #57](https://github.com/emberjs/rfcs/pull/57)
has been added in
[emberjs/data#3303](https://github.com/emberjs/data/pull/3303) and is currently
available in `ember-data` behind the `ds-references` feature flag. This already
solves some use cases, but there is still something missing.

Currently there is no public API to get notified when a relationship is
changed. Additional properties might be derived depending on the value of a
relationship. Think of a boolean indicating if comments of a post are loaded.
It is also difficult to know when the backing data of a model has been updated.

This RFC proposes new public API's (hooks on `DS.Model` similar to the ones for
`Ember.Component`) so those use cases can be implemented.


# Detailed design

<!--
This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with the framework to understand, and for somebody familiar with the implementation to implement.
This should get into specifics and corner-cases, and include examples of how the feature is used.
Any new terminology should be defined here.
-->


This RFC proposes the addition of new hooks available in `DS.Model`:

- [Relationship hooks](#relationship-hooks), which are invoked when a `belongsTo` or `hasMany` is initialized or updated
  - `didInitRelationship(name, reference)`
  - `didUpdateRelationship(name, reference)`
  - `didReceiveRelationship(name, reference)`

- [Data hooks](#data-hooks) for when the backing data of a model is modified
  - `didInitData(reference, payload)`
  - `didUpdateData(reference, payload)`
  - `didReceiveData(reference, payload)`
  - `didUpdate(reference)`

## Relationship hooks

##### didInitRelationship

```js
/**
 * The relationship was initialized for the first time.
 *
 * @param name String name of the relationship
 * @param reference {DS.BelongsToReference|DS.HasManyReference} reference to the relationship
 */
```

##### didUpdateRelationship

```js
/**
 * The relationship was updated.
 *
 * @param name String name of the relationship
 * @param reference {DS.BelongsToReference|DS.HasManyReference} reference to the relationship
 */
```


##### didReceiveRelationship

```js
/**
 * Runs in both situations (@didInitRelationship and @didUpdateRelationship).
 *
 * @param name String name of the relationship
 * @param reference {DS.BelongsToReference|DS.HasManyReference} reference to the relationship
 */
```


## Data hooks

##### didInitData

```js
/**
 * Data from the backend (via the adapter) was provided for the first time.
 *
 * @param reference DS.RecordReference reference to this model
 * @param payload Object payload from the adapter
 */
```

##### didUpdateData

```js
/**
 * Data from the backend (via the adapter) was updated.
 *
 * @param reference DS.RecordReference reference to this record
 * @param payload Object payload from the adapter
 */
```

##### didReceiveData

```js
/**
 * Runs in both situations (@didInitData and @didUpdateData).
 *
 * @param reference DS.RecordReference reference to this record
 * @param payload Object payload from the adapter
 */
```

##### didUpdate

**TODO** find different name, since there is already a `didUpdate` hook on `DS.Model`, which is invoked after a `store#updateRecord`

```js
/**
 * Invoked when either the data from the backend has changed or the data
 * has been changed locally via `store.push` or `store.pushPayload`.
 *
 * This hook can be used to update properties which are derived from
 * other attributes or relationships.
 *
 * @param reference DS.RecordReference reference to this record
 */
```

## Showcase of which hooks are invoked when

The hooks are invoked after the data has been set on the record and all
relationships are settled; similar to the existing `didUpdate`, `didLoad`, etc.
hooks.

Consider the following `post` model definition:

```js
// app/models/post.js
import Model from "ember-data/model";
import attr from "ember-data/attr";
import { belongsTo, hasMany } from "ember-data/relationships";

export default Model.extend({
  title: attr(),
  author: belongsTo(),
  category: belongsTo(),
  comments: hasMany()
});
```

To prevent unecessary materialization of records, the hooks introduced in this
RFC are only invoked on records (instances of `DS.Model`), which exist at the
time. Consider the following scenario:

```js
store.push({
  data: {
    id: 'main',
    type: 'post',
    attributes: {
      title: 'first push'
    }
  },
  included: [
    {
      id: 'sideloaded',
      type: 'post',
      attributes: {
        title: 'first push'
      }
    }
  ]
});

store.push({
  data: {
    id: 'main',
    type: 'post',
    attributes: {
      title: 'second push'
    }
  },
  included: [
    {
      id: 'sideloaded',
      type: 'post',
      attributes: {
        title: 'second push'
      }
    }
  ]
});

let mainPost = store.peekRecord('post', 'main');
let sideloadedPost = store.peekRecord('post', 'sideloaded');
```

The `didInitData` hook on `mainPost` is invoked with the state after the first
`store.push`, since the record is materialized and return from the first
`store.push`:

```js
didInitData: function() {
  assert.equal(this.get("title"), "first title");
}
```

The `didInitData` hook on `sideloadedPost` on the other hand is invoked with
the state after the second `store.push`, since the record has been materialized
after that for the first time via `store.peekRecord`:

```js
didInitData: function() {
  assert.equal(this.get("title"), "updated title");
}
```


---


The following relationship and data hooks are invoked:

```js
var post = store.createRecord('post', {
  author: 1
});

// Post#didInitRelationship: author, <BelongsToReference:author:1>
// Post#didInitRelationship: category, <BelongsToReference:category:null>
// Post#didInitRelationship: comments, <HasManyReference:comments:null>
// Post#didReceiveRelationship: author, <BelongsToReference:author:1>
// Post#didReceiveRelationship: category, <BelongsToReference:category:null>
// Post#didReceiveRelationship: comments, <HasManyReference:comments:null>

// ----------------------------------------------------------------------

post.set('author', store.peekRecord('author', 2));

// Post#didUpdateRelationship: author, <BelongsToReference:author:2>
// Post#didReceiveRelationship: author, <BelongsToReference:author:2>

// ----------------------------------------------------------------------

/**
 * POST /posts
 * {
 *   data: {
 *     id: 1,
 *     type: 'post',
 *     relationships: {
 *       author: {
 *         data: { id: 1, type: 'author' }
 *       },
 *       category: {
 *         data: { id: 1, type: 'category' }
 *       }
 *     }
 *   },
 *   included: [
 *     {
 *       id: 1,
 *       type: 'category',
 *       attributes: {
 *         label: 'uncategorized'
 *       }
 *     }
 *   ]
 * }
 */
post.save()

// Post#didInitData <RecordReference:post:1>
// Post#didReceiveData <RecordReference:post:1>
// Post#didUpdate <RecordReference:post:1>
// Post#didUpdateRelationship: category, <BelongsToReference:category:1>
// Post#didReceiveRelationship: category, <BelongsToReference:category:1>

// ----------------------------------------------------------------------

var anotherPost = store.push({
  data: {
    id: 2,
    type: 'post'
  }
});

// Post#didUpdate <RecordReference:post:2>
// Post#didInitRelationship: author, <BelongsToReference:author:null>
// Post#didInitRelationship: category, <BelongsToReference:category:null>
// Post#didInitRelationship: comments, <HasManyReference:comments:null>
// Post#didReceiveRelationship: author, <BelongsToReference:author:null>
// Post#didReceiveRelationship: category, <BelongsToReference:category:null>
// Post#didReceiveRelationship: comments, <HasManyReference:comments:null>

// ----------------------------------------------------------------------

/**
 * PUT /posts/1
 * {
 *   data: {
 *     id: 2,
 *     type: 'post',
 *     relationships: {
 *       category: {
 *         data: { id: 1, type: 'category' }
 *       }
 *     }
 *   },
 *   included: [
 *     {
 *       id: 1,
 *       type: 'category',
 *       attributes: {
 *         label: 'uncategorized'
 *       }
 *     }
 *   ]
 * }
 */
anotherPost.save()

// Post#didUpdateData <RecordReference:post:2>
// Post#didReceiveData <RecordReference:post:2>
// Post#didUpdate <RecordReference:post:2>
// Post#didUpdateRelationship: category, <BelongsToReference:category:1>
// Post#didReceiveRelationship: category, <BelongsToReference:category:1>

// ----------------------------------------------------------------------

anotherPost.get("category");

// Category#didInitData <RecordReference:category:1>
// Category#didReceiveData <RecordReference:category:1>
// Category#didUpdate <RecordReference:category:1>
```


## Use Cases

### Check if there are comments for a post without triggering a request to server

```js
// app/models/post.js
import Model from "ember-data/model";
import { hasMany } from "ember-data/relationships";

export default Model.extend({
  comments: hasMany(),

  didReceiveRelationshp(name, relationship) {
    if (name === "comments") {
      this.set("commentsLoaded", relationship.value() !== null);
    }
  }
});
```

### Get all ids without triggering a request to server

```js
// app/models/post.js
import Model from "ember-data/model";
import { hasMany } from "ember-data/relationships";

export default Model.extend({
  comments: hasMany(),

  didReceiveData(/* reference, payload */) {
    this.set('commentIds', this.hasMany('comments').ids());
  }
});

/**
 * GET /posts/1
 *
 * data: {
 *   id: 1,
 *   type: 'post',
 *   relationships: {
 *     comments: {
 *       data: [{ id: 1, type: 'comment' }]
 *     }
 *   }
 * }
 */
store.findRecord('post', 1).then(function(post) {
  assert.deepEqual(post.get("commentIds"), ["1"]);
});
```

### Update properties based on the meta of a relationship

```js
// app/models/post.js
import Model from "ember-data/model";
import { hasMany } from "ember-data/relationships";

export default Model.extend({
  comments: hasMany(),

  didReceiveData(/* reference, payload */) {
    this.set('totalComments', this.hasMany('comments').meta().total);
  }
});

/**
 * GET /posts/1
 *
 * data: {
 *   id: 1,
 *   type: 'post',
 *   relationships: {
 *     comments: {
 *       data: [{ id: 1, type: 'comment' }],
 *       meta: { total: 123 }
 *     }
 *   }
 * }
 */
store.findRecord('post', 1).then(function(post) {
  assert.deepEqual(post.get("totalComments"), 123);
});
```

# How We Teach This

<!--
What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users at any
level?

How should this feature be introduced and taught to existing Ember users?
-->

This enhances the known programming model of hooks by adding one for additional
use cases.

Since the hooks are mostly useful in combination with `ds-references`, a
dedicated blog post should make developers familiar with references and hooks.
Also the guides need to contain a detailed section outlining the use cases.

# Drawbacks

<!--
Why should we *not* do this? Please consider the impact on teaching Ember, on
the integration of this feature with other existing and planned features, on
the impact of the API churn on existing apps, etc.

There are tradeoffs to choosing any path, please attempt to identify them here.
-->

- increased API surface, as this adds 7 more hooks to the `DS.Model` API (along
the currently available `didCreate`, `didUpdate` and `didLoad` hooks)

# Alternatives

<!--
What other designs have been considered? What is the impact of not doing this?
-->

- `¯\_(ツ)_/¯`

# Unresolved questions

<!--
Optional, but suggested for first drafts. What parts of the design are still TBD?
-->

- There is currently already a `didUpdate` hook, so the one in this RFC needs to be renamed; something like `didUpdateData`?

- Should there also be `didInitAttribute`, `didUpdateAttribute` and `didReceiveAttribute` hooks?

- Is the `RecordReference` argument for the data hooks `didInitData`, `didReceiveData`, `didUpdateData` and `didUpdate` necessary?

- Which payload is passed to the data hooks? The normalized (JSON-API) or the original? Should the payload be passed at all?
