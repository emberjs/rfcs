---
Stage: Accepted
Start Date: 
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): 
RFC PR: https://github.com/emberjs/rfcs/pull/793
---

<!--- 
Directions for above: 

Stage: Leave as is
Start Date: 2022-02-11
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Ember Data
RFC PR: https://github.com/emberjs/rfcs/pull/793
-->

# Polymorphic relations without inheritance

## Summary

Allow setting polymorph relations without inheritance and mixins.

## Motivation

In pre-octane Ember you could set polymorphic relations using two ways: inheritance or mixins. Mixins are deprecated and impossible to use with native classes, so the only option is to use inheritance:

```js
import Model, { belongsTo, hasMany } from '@ember-data/model';

class TagModel extends Model {
  @belongsTo('taggable', { polymorphic: true }) taggable;
}

// not a model, just a way to make relations work
class TaggableModel extends Model {
  @hasMany('tag', { inverse: 'taggable' }) tags;
}

// real models with tags
class CommentModel extends TaggableModel {}
class PostModel extends TaggableModel {}
```

Which fells apart when you need your model to be a target of several polymorphic relations.

```js
import Model, { belongsTo, hasMany } from '@ember-data/model';

class ViewModel extends Model {
  @belongsTo('viewable', { polymorphic: true }) viewable;
}

class ViewableModel extends Model {
  @hasMany('views', { inverse: 'viewable' }) views;
}

// How to make Comment and Post viewable?...
```

You can do some dirty tricks making it inherit from each other  `ViewableModel extends TaggableModel`. Or even by adding new base model class, like this:

```js

import Model, { belongsTo, hasMany } from '@ember-data/model';

class ModelWithPolymorphic extends Model {}

class TagModel extends Model {
  @belongsTo('model-with-polymorphic', { polymorphic: true }) taggable;
}

class ViewModel extends Model {
  @belongsTo('model-with-polymorphic', { polymorphic: true }) viewable;
}

class CommentModel extends ModelWithPolymorphic {
  @hasMany('views', { inverse: 'viewable' }) views;
  @hasMany('tag', { inverse: 'taggable' }) tags;
}

```

While it solves the problem, it seems like a misuse of inheritance and too much trickery involved.

## Detailed design

I propose to introduce new API, similar to what [rails have](https://guides.rubyonrails.org/association_basics.html#polymorphic-associations).

```js
import Model, { belongsTo, hasMany } from '@ember-data/model';

class TagModel extends Model {
  @belongsTo('taggable', { polymorphic: true }) taggable;
}

class ViewModel extends Model {
  @belongsTo('viewable', { polymorphic: true }) viewable;
}

class CommentModel extends Model {
  @hasMany('views', { inverse: 'viewable' }) views;
  @hasMany('tag', { inverse: 'taggable' }) tags;
}
```

`taggable` and `viewable` in this example should be typed as `extends Model`, so any model can take that spot. 
JSON:API has enough information (`type` and `id`) to set this relation.

## How we teach this

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

The docs should be changed to promote the new way.

## Drawbacks

None

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Questions

It might be considered to use `as` instead of `inverse` to make it stand out as polymorphic relation, but if it works without it, I think it's better to keep `inverse`.
