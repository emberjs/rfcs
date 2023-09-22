---
stage: released
start-date: 2021-04-23T00:00:00.000Z
release-date:
release-versions:
teams:
  - data
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/740'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/910'
  released: 'https://github.com/emberjs/rfcs/pull/967'
project-link:
---

# EmberData | Deprecate Non Strict Types

## Summary

Deprecates when the `type` for a record provided by a user differs from the resolved
type, thereby removing the need to configure `ember-inflector` to treat `types` as `uncountable`
in order to use plural model names, and removing the dasherization constraint.

## Motivation

Today, `ember-data` normalizes user supplied `type` or `modelName` anywhere it is encountered
so that it will match the `type` we expect to provide to ember's resolver to lookup and create
instances of `Model`/`Adapter`/`Serializer`.

In practice, this resulted in a convention of singularized, dasherized modelName arguments and
`type` fields in payloads. However, this convention is unimportant to how EmberData operates and
we feel that users should be able to name their models however they like provided that (1) they
use their format consistently and (2) that format matches what ember's resolver needs to resolve
any necessary modules (such as models).

Today, if you wanted to name a model `posts`, you could achieve this by

- explicitly definining the inverse type on all relationships pointing at `posts`
- using `posts` as the type in all payloads provided to the store
- naming your model on disk as `models/posts.js`
- configuring `ember-inflector` to treat `posts` as either uncountable or as it's own singular/plural.

While the first three items here are strict conventions, we would like to do away with the necessity of
configuring `ember-inflector` in this manner.

Similarly, if you want to name your models with camelCase or SnakeCase types instead of dasherized types
we see no reason to enforce that you do otherwise.

In this sense the name of this RFC may at first feel in opposition to the motivation: we are proposing
deprecating non-strict usage of types so that we can loosen restrictions on type usage.

By removing support for non-strict we *strictly* mean removing support for using types in a way that
depends upon our normalization in order for resolver lookups to succeed.

## Detailed design

We would print a deprecation whenever our normalization results in a different string than the string originally
received. This deprecation would target `5.0` and become `enabled` no-sooner than `4.1` although it may be made
`available` before then.

To resolve issues with `dasherization`, users would need to dasherize in advance of providing data or arguments
to `store` methods (generally this is done in the serializer's normalization hooks for payloads and at
call-sights for method args that take `modelName`).

To resolve issues with `singularization`, users would need either to configure `ember-inflector` to return the
desired form for their string if the supplied string is the desired singular/plural, or singularize the string
in advance.

Once a user has resolved this deprecation and marked their app as compatible with the version of `ember-data`
in which this deprecation became enabled all support for ember-data's normalization will be removed at build
time and users may supply types in any format so long as their usage is (1) consistent and (2) resolveable.

## How we teach this

Generally these changes remove a thing to be taught (that all types anywhere in a payload or as args should
be normalized to the singular+dasherized form) in favor of teaching that usage of a `type` should be consistent
and that the name of the `model` should match what is being used as `type`.

## Drawbacks

Potentially some churn if it turns out that lots of users rely on the dasherization aspect. We already stopped
singularizing on our own some time ago except in the case of hasMany relationship definitions. We can mitigate
this to a good degree by making sure that the default serializers dasherize their output if they do not already.

## Alternatives

Continue to enforce dasherized-singular types and pay the cost of requiring use of `ember-inflector` if using
`ember-data`.
