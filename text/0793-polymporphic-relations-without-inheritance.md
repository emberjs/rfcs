---
stage: accepted # FIXME: This may be a further stage
start-date: 2022-02-11T00:00:00.000Z
release-date:
release-versions:
teams:
  - data
prs:
  accepted: https://github.com/emberjs/rfcs/pull/793
project-link:
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

The polymorphic relationship **MUST** explicitly declare itself as polymorphic and **MUST** explicitly declare it's inverse as either a key or `null`. All records that satisfy the polymorphic type must declare a matching inverse via that key.

Polymorphic relationships take the form:

```ts
hasMany(abstractType: string, options: { async: boolean, polymorphic: true, inverse: string | null })
belongsTo(abstractType: string, options: { async: boolean, polymorphic: true, inverse: string | null })
```

Concrete Relationships which fulfill polymorphic relationships take the form below where the string supplied to `as` should be the same `abstractType` supplied to the polymorphic definition above.

```ts
hasMany(recordWithPolymorphicRelationship: string, options: { async: boolean, inverse: string, as: string })
belongsTo(recordWithPolymorphicRelationship: string, options: { async: boolean, inverse: string, as: string })
```

So for instance, given a field named "tagged" that is a polymorphic hasMany with baseType "taggable", and inverse key "tags" the following would be the polymorphic relationship and concrete relationship.

```ts
// polymorphic relationship
class Tag extends Model {
   @hasMany("taggable", { async: false, polymorphic: true, inverse: "tags" }) tagged;
}
// an inverse concrete relationship
class Post extends Model {
   @hasMany("tag", { async: false, inverse: "tagged", as: "taggable" }) tags;
}
```

### Polymorphic Relationships without Inverses

- polymorphic relationships need not have an inverse, in which case they must specify `inverse: null`.

### Base Types

Every polymorphic relationship has an implicit "abstract type" that represents the entity on the other side of the relationship.

In the below example, the polymporphic relationship implies the existence of the `abstract type` "taggable".

```ts
class Tag extends Model {
  @hasMany("taggable", { async: false, inverse: "tags", polymorphic: true }) taggables;
}
```

It is only required that your app implement a model (or if using instantiateRecord and the schema service implement handling for that type) if your API will return entities whose type is the abstract type. If the abstract type is never directly interacted with, it is not required to implement a Model or other associated support.

### Polymorphic to Polymorphic Relationships

A Polymorphic to Polymorphic Relationship can be declared by both sides of the relationship declaring both `polymorphic: true` and `as: <abstract-type>`. For instance taggables that could also be editables might be implemented like below:

```ts
class Tag extends Model {
   @hasMany('taggable', { async: false, polymorphic: true, inverse: 'tags', as: 'editable' }) tagged;
}

class Post extends Model {
   @hasMany('editable', { async: false, polymorphic: true, inverse: 'tagged', as: 'taggable' }) tags;
}
```

### Schemas for Abstract Types

When no model exists for the abstract type EmberData has no mechanism for validating whether the
 associated relationship is single-resource relationship (belongsTo) or a collection relationship (hasMany). The simpest way when using `@ember-data/model` to ensure this information is available is to implement the abstract model as well; however, this is not the *best* mechanism.

The best way to inform the store of the shape of the relationship on the abstract type is to supply
 the information via the `SchemaDefinitionService`. When not using `@ember-data/model` this service is always supplied by the app or by another addon (for instance `ember-m3`). That implementation should be designed such that it responds to queries for schema for the abstract type with the appropriate information.

When using `@ember-data/model`, you can use the delegator pattern to achieve this. Below we show the `taggable` and `editable` abstract types from our last example above being handled in this way.

```ts
import Store from '@ember-data/store';

// Relationship Schemas for the abstract types
// An app could hard code this, import it, or load it as needed
// via their API.
const AbstractSchemas = new Map([
  [
    'taggable',
    {
      tags: {
        kind: 'hasMany',
        type: 'editable',
        name: 'tags',
        options: {
          async: false,
          polymorphic: true,
          inverse: 'tagged',
          as: 'taggable'
        }
      }
    }
  ],
  [
    'editable',
    {
      tagged: {
        kind: 'hasMany',
        type: 'taggable',
        name: 'tagged',
        options: {
          async: false,
          polymorphic: true,
          inverse: 'tags',
          as: 'editable'
        }
      }
    }
  ],
]);

class SchemaDelegator {
  constructor(schema) {
    this._schema = schema;
  }

  doesTypeExist(type: string): boolean {
    if (AbstractSchemas.has(type)) {
      return false; // some apps may want `true`
    }
    return this._schema.doesTypeExist(type);
  }

  attributesDefinitionFor(identifier: RecordIdentifier | { type: string }): AttributesSchema {
    return this._schema.attributesDefinitionFor(identifier);
  }

  relationshipsDefinitionFor(identifier: RecordIdentifier | { type: string }): RelationshipsSchema {
    const schema = AbstractSchemas.get(identifier.type);
    return schema || this._schema.relationshipsDefinitionFor(identifier);
  }
}

export default class extends Store {
  constructor(...args) {
    super(...args);

    const schema = this.getSchemaDefinitionService();
    this.registerSchemaDefinitionService(new SchemaDelegator(schema));
  }
}
```

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

## Questions

None
