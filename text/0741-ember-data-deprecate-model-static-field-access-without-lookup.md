---
start-date: 2021-04-23T00:00:00.000Z
release-date:
release-versions: 
teams: 
  - data
prs:
  accepted: https://github.com/emberjs/rfcs/pull/741
project-link: 
stage: accepted
---

# EmberData | Deprecate Accessing Static Fields On Model Prior To Lookup

## Summary

Deprecates when a user directly imports a model extending `@ember-data/model` and
attempts to access one of the static fields containing schema information about
attributes or relationships.

## Motivation

Schema descriptors on classes extending `@ember-data/model` require walking the prototype
chain to collect inherited attributes and relationships. This isn't possible until an
`EmberObject`'s private `proto` method has been invoked.

Externally, we feel accessing schema information in this manner is a bad practice that should
be avoided. Schema information is exposed at runtime via `store.modelFor` in pre-
[custom-model-class](https://github.com/emberjs/rfcs/blob/master/text/0487-custom-model-classes.md#custom-model-class-rfc)
versions and via the schema definition service and store-wrapper in post- custom-model-class
versions (3.28+).

Internally, the EmberData team wishes to explore removing our dependence on `computed`
properties and `eachComputedProperty` which make use of invoking `proto` for defining
schema (`attr|belongsTo|hasMany` currently utilize these APIs to build out the schema information).

## Detailed design

If we detect that an access has been made on a class not provided by a factory (the result of
calling `modelFor`) we would print a deprecation targeting `5.0` that would become `enabled`
no-sooner than `4.1` (although it may be made `available` before then).

Most usages of this pattern occur when a user imports a model for a unit test. In these cases
the appropriate refactor to look it up via the store like so:

```js
test('my test', async function(assert) {
  const UserSchema = this.owner.lookup('service:store').modelFor('user');
  let { attributes } = UserSchema; // access the map of attributes
});
```

Or if defining the model in the test file, first register the model like so:

```js
test('my test', async function(assert) {
  class User extends Model {
    @attr name;
  }
  this.owner.register('model:user', User);

  const UserSchema = this.owner.lookup('service:store').modelFor('user');
  let { attributes } = UserSchema; // access the map of attributes
});
```

## How we teach this

Generally this pattern has not been widely observed, though we should make sure that api-docs,
guides, and blueprints are all updated to show the preferred pattern.

## Drawbacks

Potentially some churn if it turns out that lots of users rely on this sort of access, though
generally this is just another clear step away from `EmberObject` and it better prepares these
users for accessing schema on models not built off of `@ember-data/model`.

## Alternatives

Ignore that this pattern exists, which seems risky given our momentum towards custom model
classes and potentially a different default model than `@ember-data/model`.
