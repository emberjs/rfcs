- Start Date: 2014-08-20
- RFC PR:
- Ember Issue:

## Summary

`pathTemplate` improves the extesibility of API endpoint urls in `RESTAdapter`.

## Motivation

I think that Ember Data has a reputation for being hard to configure. I've often
heard it recommended to design the server API around what Ember Data expects.
Considering a lot of thought has gone in to the default RESTAdapter API, this
is sound advice. However, this is a false and damaging reputation. The adapter
and serializer pattern that Ember Data uses makes it incredibly extensible. The
barrier of entry is high though, and it's not obvious how to get the url you need
unless it's [a namespace](http://emberjs.com/guides/models/connecting-to-an-http-server/#toc_url-prefix)
or something [pathForType](http://emberjs.com/guides/models/customizing-adapters/#toc_path-customization)
can handle. Otherwise it's "override `buildURL`". `RESTSerializer` was recently
improved to make handling various JSON structures easier; it's time for url
configuration to be easy too.

## Detailed Design

`buildURL` and associated methods and properties will be moved to a mixin design
to handle url generation only. `buildURL` will use templates to generate a URL
instead of manually assembling parts. Simple usage example:

```javascript
export default DS.RESTAdapter.extend({
  namespace: 'api/v1',
  pathTemplate: '/:namespace/posts/:id'
});
```

### Resolving template segments

Each dynamic path segment will be resolved on a singleton object based on a `pathSegments`
object provided in the adapter. This `pathSegments` object will feel similar to defining
actions on routes and controllers.

```javascript
// adapter
export default DS.RESTAdapter.extend({
  namespace: 'api/v1',
  pathTemplate: '/:namespace/posts/:post_id/:category_name/:id',

  pathSegments: {
    category_name: function(record) {
      return _pathForCategory(record.get('category'));
    }

    post_id: function(record) {
      return record.get('post.id');
    };
  }
});
```

#### Psuedo implementation

```javascript
RESTAdapter = Adapter.extend({
  buildURL: function(type, id, record) {
    var urlResolver = _lookupURLResolver(type);
    var template = this.get('pathTemplate')
    var urlParts = _parseURLTemplate(template, function(name) {
      var fn = urlResolver.get(name);
      return fn(record);
    });

    return this.urlPrefix(urlParts.compact().join('/'));
  }
});

function _lookupURLResolver(type) {
  // a singleton object will be created using pathSegments that includes important
  // adapter attributes (such as namespace and host) and delegates unknown
  // properties to the record, with something like:
  //
  //  unknownProperty: function(key) {
  //    return function(record) { return record.get(key); };
  //  }
};

// This is the simplest
function _parseURLTemplate(template, fn) {
  var parts = template.split('/');
  return parts.map(function(part) {
    if (_isDynamicSegment(part)) { // if segment starts with a `:`
      return fn(_segmentName(part)); // strip off the `:`
    } else {
      return part;
    }
  });
};
```

### Different URL templates per action

Configuring different urls for different actions becomes fairly trivial with this
system in place:

#### Usage

```javascript
// adapter
export default DS.RESTAdapter.extend({
  namespace: 'api/v1',
  pathTemplate: '/:namespace/posts/:post_id/:id',
  createPathTemplate: '/:namespace/comments',
  hasManyTemplate: '/:namespace/posts/:post_id/comments',

  pathSegments: {
    post_id: function(record) {
      return record.get('post.id');
    };
  }
});
```

#### Psuedo implementation

```javascript
RESTAdapter = Adapter.extend({
  createRecord: function(store, type, payload) {
    // ...
    var url = this.buildURL(type.typeKey, null, record, 'create');
    return this.ajax(url, "POST", { data: data });
  },

  buildURL: function(type, id, record, action) {
    var template = this.get(action + 'PathTemplate') || this.get('pathTemplate');
    // ...
  }
});
```

## Drawbacks

* Building URLs in this way is likely to be less performant. If this proposal is
  generally accepted, I will run benchmarks.

## Alternatives

The main alternative that comes to mind, that would make it easier to configure
urls in the adapter, would be to generally simplify `buildURL` and create more
hooks.

## Unresolved Questions

* How many templates are reasonable? There could be templates for different
  operations such as `createRecord`, `updateRecord`, but also `findQuery`, etc.
* How do we handle generating urls for actions that do not have a single
  record? This includes `findAll` and `findQuery`, which have no record, and
  `findMany` and `findHasMany` which have a collection of records.

