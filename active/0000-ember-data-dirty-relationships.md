- Start Date: 2014-11-08
- RFC PR:
- Ember Issue:

# Summary

Provide a way to mark a model as dirty when relationships or related model properties change.

# Motivation

There is a common pattern to conditionally show a save button if a model needs saving.

For most cases, using the `isDirty` flag on the model is enough to determine
whether a model needs saving or not. However, this flag currently ignores relationships,
making it unusable for anything other than simple properties.

Ideally, the user would be able to specify which relationships are saved when the model
is saved, and thus, which ones should mark the model as being dirty.

# Detailed design

In order to prevent dirtiness from spreading across the entire object graph,
dirtying would have to be opt in. Users would specify on each relationship
whether it would dirty the model or not.

```js
var Comment = DS.Model.extend({
  post: DS.belongsTo('post', { dirties: true }),
  user: DS.belongsTo('user')
});

var comment = this.store.find('comment', 1);
comment.get('post');    // null
comment.get('user');    // null
comment.get('isDirty'); // false

comment.set('user', someUser);
comment.get('isDirty'); // false

comment.set('post', somePost);
comment.get('isDirty'); // true

comment.set('post', null);
comment.get('isDirty'); // false
```

For `hasMany` relationships, adding or removing a relationship would dirty the model.

```js
var Post = DS.Model.extend({
  tags:     DS.hasMany('tag', { dirties: true }),
  comments: DS.hasMany('comment')
});

var post = this.store.find('post', 1);
post.get('isDirty');         // false
post.get('tags.length');     // 0
post.get('comments.length'); // 0

post.get('comments').pushObject(someComment);
post.get('comments.length'); // 1
post.get('isDirty');         // false

post.get('tags').addObject(someTag);
post.get('tags.length'); // 1
post.get('isDirty');     // true

post.save();
post.get('isDirty'); // false

post.get('tags').removeObject(someTag);
post.get('tags.length'); // 0
post.get('isDirty');     // true
```

Sometimes, entire related models are serialized when a model is saved.
In those circumstances, it would be useful to allow the model to be dirtied
when any of the related model properties are saved.

```js
var User = DS.Model.extend({
  address: DS.belongsTo('address', { dirties: 'properties' })
});

var Address = DS.Model.extend({
  street: DS.attr('string')
});

var user = this.store.find('user', 1);
user.get('isDirty'); // false
user.get('address.street'); // 4th Ave

user.set('address.street', 'Main Street');
user.get('isDirty'); // true

user.set('address.street', '4th Ave');
user.get('isDirty'); // false
```

# Drawbacks

This would significantly increase the complexity of the `isDirty` computed property.

There would probably need to be checks in place to prevent recursive dirty checking.
In the example below, once either of the models is made dirty, they would never become
clean again by reverting a property, as each one's `isDirty` state is dependent on the other's.

```js
var A = DS.Model.extend({
  b: DS.belongsTo('b', { dirties: true })
});
var B = DS.Model.extend({
  a: DS.belongsTo('a', { dirties: true })
});
```

# Alternatives

[Paul Ferrett's isFilthy computed property](http://paulferrett.com/2014/ember-model-isdirty-when-belongsto-changes/)

[Steven Lindberg's Dependent Relationships](https://gist.github.com/slindberg/8660986)

[Tim Wood's Dirty Relationships Mixins](https://gist.github.com/timrwood/c58b667650bee749d5af)

# Unresolved questions

The syntax feels like it could use some bikeshedding.

```js
DS.belongsTo('other', { dirtiesOn: 'relationship' })
DS.belongsTo('other', { dirtiesOn: 'properties' })
```

Conceptually, a relationship should be considered dirty if the relationship is being
saved by the model, so it could be named something related to saving.

```js
DS.belongsTo('other', { savesRelation: true })
DS.belongsTo('other', { savesModel: true })
```
