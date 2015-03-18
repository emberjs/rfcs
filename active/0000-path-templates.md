- Start Date: 2014-08-20
- RFC PR:
- Ember Issue:

## Summary

`url-templates` improves the extesibility of API endpoint urls in `RESTAdapter`.

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
  urlTemplate: '{host}{/namespace}/posts{/id}'
});
```

### Resolving template segments

Each dynamic path segment will be resolved on a singleton object based on a `urlSegments`
object provided in the adapter. This `urlSegments` object will feel similar to defining
actions on routes and controllers.

```javascript
// adapter
export default DS.RESTAdapter.extend({
  namespace: 'api/v1',
  pathTemplate: '{/:namespace}/posts{/:post_id}{/:category_name}{/:id}',

  pathSegments: {
    category_name: function(type, id, snapshot, requestType) {
      return _pathForCategory(snapshot.get('category'));
    }

    post_id: function(type, id, snapshot, requestType) {
      return snapshot.get('post.id');
    };
  }
});
```

#### Psuedo implementation

```javascript
export default Adapter.extend({
  buildURL: function(type, id, record, requestType) {
    var template = this.compileTemplate(this.get('urlTemplate'));
    var templateResolver = this.templateResolverFor(type);
    var adapter = this;

    return template.fill(function(name) {
      var result = templateResolver.get(name);

      if (Ember.typeOf(result) === 'function') {
        return result.call(adapter, type, id, record, requestType);
      } else {
        return result;
      }
    });
  },
});
```

### Different URL templates per action

Configuring different urls for different actions becomes fairly trivial with this
system in place:

#### Usage

```javascript
// adapter
export default DS.RESTAdapter.extend({
  urlTemplate: '/posts{/:post_id}{/:id}',
  createUrlTemplate: '/comments',
  hasManyUrlTemplate: '/posts/{:post_id}/comments',

  pathSegments: {
    post_id: function(snapshot) {
      return snapshot.get('post.id');
    };
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

* How do we handle generating urls for actions that do not have a single
  record? This includes `findAll` and `findQuery`, which have no record, and
  `findMany` and `findHasMany` which have a collection of records.

