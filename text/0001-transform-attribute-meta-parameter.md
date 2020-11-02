---
Start Date: 2014-08-14
RFC PR: https://github.com/emberjs/rfcs/pull/1
Ember Issue: https://github.com/emberjs/data/pull/4086

---

# Summary

For Ember Data. Pass through attribute meta data, which includes `parentType`, `options`, `name`, etc.,
to the transform associated with that attribute. This will allow provide the following function signiture updates to `DS.Transform`: 

* `transform.serialize(deserialized, attributeMeta)`
* `transform.deserialize(serialized, attributeMeta)`

# Motivation

The main use case is to be able to configure the transform
on a per-model basis making more DRY code. So the transform can be aware of type and options on `DS.attr` can
be useful to configure the transform for DRY use.

# Detailed design

## Implementing

The change will most likely start in [`eachTransformedAttribute`][1], which gets the attributes for that instance via `get(this, 'attributes')`. In the `forEach` the `name` will be used to get the specific attribute, e.g.

```js
var attributeMeta = attributes.get(name);
callback.call(binding, name, type, attributeMeta);
```

The next change will be in [`applyTransforms`][2], where the `attributeMeta` parameter is added and passed to `transform.deserialize` as the second argument.

You also have to handle the serialization part in [`serializeAttribute`][3], where you pass through the `attribute` parameter to `transform.serialize`.

## Using

A convoluted example:

```js
// Example based on https://github.com/chjj/marked library
App.PostModel = DS.Model.extend({
  title: DS.attr('string'),
  markdown: DS.attr('markdown', {
    markdown: {
      gfm: false,
      sanitize: true
    }
  })
});

App.TechnicalPostModel = DS.Model.extend({
  title: DS.attr('string'),
  gistUrl: DS.attr('string'),
  markdown: DS.attr('markdown', {
    markdown: {
      gfm: true,
      tables: true,
      sanitize: false
    }
  })
});

App.MarkdownTransform = DS.Transform.extend({
  serialize: function (deserialized, attributeMeta) {
    return deserialized.raw;
  },
  
  deserialize: function (serialized, attributeMeta) {
    var options = attributeMeta.options.markdown || {};
    
    return marked(serialized, options);
  }
});
```

# Drawbacks

Extra API surface area, although not much. This could also potentially introduce tight coupling between models and transforms if used improperly, e.g. not returning a default value if using type checking.

# Alternatives

1. Passing the information from the server, which is a poor solution.
2. Writing a new transform for each model/attribute that needs a variation. Although this might be a good solution sometimes if you extend a base transform.

# Unresolved questions

Does the whole meta object need to be passed, or do we selectively pass in only the useful properties? Like
`options` and `parentType` and `name`..



[1]: https://github.com/emberjs/data/blob/master/packages/ember-data/lib/system/model/attributes.js#L193
[2]: https://github.com/emberjs/data/blob/master/packages/ember-data/lib/serializers/json_serializer.js#L117
[3]: https://github.com/emberjs/data/blob/master/packages/ember-data/lib/serializers/json_serializer.js#L528
