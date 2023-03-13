---
stage: ready-for-release
start-date: 2021-04-23T00:00:00.000Z
release-date:
release-versions:
teams:
  - data
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/739'
project-link:
---

# EmberData | Deprecate Non Strict Relationships

## Summary

Deprecates various shorthands for defining a `belongsTo` or `hasMany`
relationship that create ambiguity, cause expensive runtime lookups,
or hinder static analysis.

## Motivation

Deprecating these shorthands allows us to provide better tooling, remove the bulky
and expensive code necessary to determine the outcomes of these shorthands at
runtime, and in the process clarify and simplify relationship behaviors in the
documentation. Further, it allows us to align our own internal usage of
"relationship meta" with the API put forward in [RFC#487 Custom Model Classes](https://github.com/emberjs/rfcs/blob/master/text/0487-custom-model-classes.md#exposing-schema-information).

## Detailed design

Three shorthands are proposed to be deprecated, with the deprecation targeting `5.0`
and being `enabled` no-sooner than `4.1`. It may be `available` sooner.

- Deprecates the relationship shorthand that does not provide the related type as the first argument.

This deprecation is related to deprecating `non-strict` types as it requires us to
normalize the property name into a type string in a manner which may not be correct.

*Before*

```./before.js
import Model, { belongsTo, hasMany } from '@ember-data/model';

export default class Comment extends Model {
  @belongsTo()
  author;

  @hasMany({ async: true, inverse: null })
  likes;
}
```

*After*

```./after.js
import Model, { belongsTo, hasMany } from '@ember-data/model';

export default class Comment extends Model {
  @belongsTo('author')
  author;
  @hasMany('like', { async: true, inverse: null })
  likes;
}
```

- Deprecates not explicitly setting the relationship inverse property (or `null`).

The current default when not specified is to attempt to find a property on the
related model that either explicity or implicitly points back, resolving to `null`
if no such relationship is found and erring if more than one potential inverse was
discovered.

*Before*

```./before.js
import Model, { belongsTo, hasMany } from '@ember-data/model';

class Comment extends Model {
  @belongsTo('user')
  author;

  @hasMany('like')
  likes;
}

class User extends Model {
  @hasMany('comment')
  comments;
}

class Like extends Model {
  @belongsTo('user')
  author;
}
```

*After*

```./after.js
import Model, { belongsTo, hasMany } from '@ember-data/model';

class Comment extends Model {
  @belongsTo('user', { inverse: 'comments' })
  author;

  @hasMany('like', { inverse: null })
  likes;
}

class User extends Model {
  @hasMany('comment', { inverse: 'author' })
  comments;
}

class Like extends Model {
  @belongsTo('user', { inverse: null })
  author;
}
```

- Deprecates not explicitly setting `async` to `true|false`.

The current default when not specified is `true`.

*Before*

```./before.js
import Model, { belongsTo, hasMany } from '@ember-data/model';

class Comment extends Model {
  @belongsTo('user', { inverse: 'comments' })
  author;

  @hasMany('like', { inverse: null })
  likes;
}

class User extends Model {
  @hasMany('comment', { inverse: 'author' })
  comments;
}

class Like extends Model {
  @belongsTo('user', { inverse: null })
  author;
}
```

*After*

```./after.js
import Model, { belongsTo, hasMany } from '@ember-data/model';

class Comment extends Model {
  @belongsTo('user', { async: true, inverse: 'comments' })
  author;

  @hasMany('like', { async: true, inverse: null })
  likes;
}

class User extends Model {
  @hasMany('comment', { async: true, inverse: 'author' })
  comments;
}

class Like extends Model {
  @belongsTo('user', { async: true, inverse: null })
  author;
}
```

## Codemod

A codemod should be provided. This codemod would analyze a user's relationships
and wherever possible format the provided options with the now deprecated missing
information. Note: For the related type and inverse property name when
mixin-based polymorphism is present codemods may not be practical.

## How we teach this

Generally these changes improve our ability to document and explain relationship
APIs as confusing behaviors (such as undefined `async` defaulting to `true`) and
inverse determination are no longer magical resolutions but explicit declarations.
API documentation will be updated and any examples in the guides should be as well.

## Drawbacks

Some churn, but via codemod we can make this quick and seamless for most apps.

## Alternatives

Provide new decorators that replace these. Leave these alone. While we may introduce
new decorator primitives in the near future, iterating on the existing decorators to
make them stricter in largely painless ways will allow more teams to migrate to a full
replacement in the future with greater ease.
