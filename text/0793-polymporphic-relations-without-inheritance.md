---
start-date: 2022-02-11T00:00:00.000Z
release-date:
release-versions: 
teams: 
  - data
prs:
  accepted: https://github.com/emberjs/rfcs/pull/793
project-link: 
stage: accepted
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

# EmberData | Polymorphic Relationship Support

## Summary

1. Allow setting polymorphic relationships without inheritance and mixins.
2. Deprecate using inheritance and mixins for polymorphic relationships. Target version: 5.0.

## Motivation

Currently, although undocumented and not explicitly supported, you can set polymorphic
relationships using two mechanisms: inheritance and mixins. This RFC seeks to lay out
an explicitly supported path for polymorphic relationships that composes well with
native class usage, eliminates the dependency on instanceof checks and prototype chain
walking, and is compatible with the schema service introduced in [RFC #487](https://github.com/emberjs/rfcs/blob/master/text/0487-custom-model-classes.md#exposing-schema-information)

**Example Using Inheritance**

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

**Example via Mixins**

Consider that this same post and comment class also both need to be "viewable", and that "viewable" is a trait also shared by other models that are not "taggable". Today this is only achievable via mixins.

```js
import Model, { belongsTo, hasMany } from '@ember-data/model';
import Mixin from "@ember/object/mixin";

class TaggableMixin extends Mixin {
  @belongsTo('taggable', { polymorphic: true }) taggable;
}

class ViewableMixin extends Mixin {
  @belongsTo('viewable', { polymorphic: true }) viewable;
}

class CommentModel extends Model.extends(TaggableMixin, ViewableMixin) {
  @hasMany('tag', { inverse: 'taggable' }) tags;
  @hasMany('views', { inverse: 'viewable' }) views;
}

```

## Detailed design

This RFC proposes to allow explicitly declaring polymorphic traits similar to [rails conventions](https://guides.rubyonrails.org/association_basics.html#polymorphic-associations).

### API

The inverse relationship, if one exists, of a polymorphic relationship must be concrete (e.g. not also polymorphic).

The polymorphic relationship **MUST** explicitly declare itself as polymorphic and **MUST** explicitly declare it's inverse as either a key or `null`. All records that satisfy the polymorphic type must declare a matching inverse via that key.

Polymorphic relationships take the form:

```ts
hasMany(baseType: string, options: { polymorphic: true, inverse: string | null })
belongsTo(baseType: string, options: { polymorphic: true, inverse: string | null })
```

Concrete Relationships which fulfill polymorphic relationships take the form:

```ts
hasMany(recordWithPolymorphicRelationship: string, options: { inverse: string, as: string })
belongsTo(recordWithPolymorphicRelationship: string, options: { inverse: string, as: string })
```

So for instance, given a field named "tagged" that is a polymorphic hasMany with baseType "taggable", and inverse key "tags" the following would be the polymorphic relationship and concrete relationship.

```ts
// polymorphic relationship
class Tag extends Model {
   @hasMany("taggable", { polymorphic: true, inverse: "tags" }) tagged;
}
// an inverse concrete relationship
class Post extends Model {
   @hasMany("tag", { inverse: "tagged", as: "taggable" }) tags;
}
```

### Polymorphic Relationships without Inverses

- polymorphic relationships need not have an inverse, in which case they must specify `inverse: null`.

### Base Types

Every polymorphic relationship has an implicit "base type" that represents the entity on the other side of the relationship.

In the below example, the polymporphic relationship implies the existence of the `base type` "taggable".

```
class Tag extends Model {
  @hasMany("taggable", { inverse: "tags", polymorphic: true }) taggables;
}
```

It is only required that your app implement a model (or if using instantiateRecord and the schema service implement handling for that type) if your API will return entities whose type is the base type. If the base type is abstract and never directly interacted with, it is not required to implement a Model or other associated support.

### Limitations

- polymorphic to polymorphic relationships are not allowed. One side of the relationship MUST be a concrete entity type or have no inverse.
- this includes self-referential relationships

### Deprecation Design

Situations in which polymorphism is not configured by the above mechanism but in which EmberData would have previously detected and attempted to do the right thing will be deprecated. This means effectively that mixin based and inheritance based polymorphism will print a deprecation *only when* the corresponding relationships are not also configured for polymorphism correctly. By this means users are free to continue using mixins and inheritance if they so choose.

## How we teach this

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

Polymorphism was never explicitly documented or supported by EmberData. With this RFC, we would add an official declaration of support and add a section on polymorphism to both the docs for model, docs for the schema service and to the guides.

## Drawbacks

None

## Alternatives

We could require a more explicit API that declares the full-intent and full-resolvability on one or even both sides. The advantage of both sides is both less work to do on the part of the ember-data to determine validity and increased clarity to the reader in that both sides show the config.

However, This can have some drawbacks in that as an API grows to have more entities that satisfy a polymorphic relationship the config becomes increasingly and unnecessarily large and prevents runtime additions to the schema.

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Questions

None
