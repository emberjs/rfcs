- Start Date: 2018-20-08
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# [Ember Data] Improve findHasMany/findBelongsTo decision making

## Summary

Expose the functionality to decide how to load a relationship (via `links` or via `data`).

## Motivation

Currently, Ember Data decides, based on some internal criteria, if it should use the `links` through the `findHasMany`/`findBelongsTo` methods on the adapter, or the included `data`, to load a relationship. While this generally works, it is not (easily) possible to customize this behavior for special use cases.

## Detailed design

I propose to add a new public function to the REST Adapter: `shouldFindViaLink`. This basically just does the same thing that currently happens inside of the private, undocumented `_findHasManyByJsonApiResource`/`_findBelongsToByJsonApiResource` methods on the store service. The default implementation would look like this:

```js
shouldFindViaLink(resource, relationshipType, parentInternalModel, relationshipMeta, options) {
    let {
      relationshipIsStale,
      allInverseRecordsAreLoaded,
      hasDematerializedInverse,
      relationshipIsEmpty,
    } = resource._relationship;

    let hasLink = !!(resource.links && resource.links.related);

    let shouldLoad = (hasDematerializedInverse ||
      relationshipIsStale ||
      (!allInverseRecordsAreLoaded && !relationshipIsEmpty));

    return hasLink && shouldLoad;
}
```

This function would then be called by both the `findHasManyByJsonApiResource` as well as the `findBelongsToByJsonApiResource` methods on the store service:

```js
function _findHasManyByJsonApiResource(resource, parentInternalModel, relationshipMeta, options) {
    if (!resource) {
      return RSVP.resolve([]);
    }

    let modelName = relationshipMeta.meta.type;
    let adapter = this.adapterFor(modelName);

    let shouldFindViaLink = adapter.shouldFindViaLink(resource, 'hasMany', parentInternalModel, relationshipMeta, options);

    // etc.
}
```

Now, users can overwrite the `shouldFindViaLink` in their adapter, to accomodate special use cases.

### Example A

This would make it easy to force Ember Data to use the `findHasMany`/`findBelongsTo` methods for relationships that include neither `data` nor `links`:

```js
shouldFindViaLink(resource) {
    let { data, links } = resource;

    if (!data && !links } {
        return true;
    }

    return this._super(...arguments);
}
```

### Example B

This also allows specifiying to e.g. use `data` in certain cirumstances, for example when the relationship was never loaded, and `links` for reloading relationships:

```js
shouldFindViaLink(resource, relationshipType, parentInternalModel, relationshipMeta) {
    let record = parentInternalModel._record;
    let ref = record[relationshipType](relationshipMeta.name);

    if (resource.data && ref.value() === null) {
      return false;
    }

    return this._super(...arguments);
}
```

## How we teach this

This change would be completely backwards compatible (including private APIs), and only add new customization options. We might want to provide usage examples (as the ones provided in this RFC). 

## Drawbacks

It increases the API surface of the adapter.

## Alternatives

We could also split up the `_findHasManyByJsonApiResource`/`_findBelongsToByJsonApiResource` methods on the store to make them easier to overwrite. However, I do think that this is better suited to live on the adapter.

We could also change the existing `findHasMany`/`findBelongsTo` methods on the adapter and move the decision making in these functions. However, I do think this might become tricky, as it will probably require bigger changes to the `findHasMany`/`findBelongsTo` methods - as it returns an Ajax promise, we'd need to keep this function signature.

Finally, we could also introduce new functions (like `loadHasManyRelationship`/`loadBelongsToRelationship`) and put the decision making in there. This would potentially increase the already existing confusion about what is `findMany`, `findHasMany`, `loadHasManyRelationships` etc. 

## Unresolved questions

Which exact arguments should the `shouldFindViaLink` method receive? I uses the ones used in the private `_findHasManyByJsonApiResource` method, plus a relationship type to make it work for both has many & belongs to relationships. 
