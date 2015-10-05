- Start Date: 2015-10-11
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes allowing arbitrary query parameters to be specified when finding
records using the Ember Data store in order to improve support for the [JSON
API][json-api] spec.

# Motivation

While support for JSON API has largely improved in Ember Data there are still a
few areas of the specification that aren't easy to make use of. If we're going
to push for JSON API to be used, it's important that Ember Data makes using
features such as [including related resources][fetching-includes] and [fetching sparse
fieldsets][fetching-sparse-fieldsets] as easy as possible.

## Why not use query and queryRecord?

When using `store.query()` and `store.queryRecord()` you are forced into always
making a request to the server to fetch data before proceeding. You don't have
the option to use what's currently in the store right away and load fresh data
in the background. This makes sense if what you're doing is truly some kind of
search query that would need logic to be provided in Ember to also provide
accurate results but that is not always the case.

# Detailed design

In short, I think it would be useful to allow a `query` object to be provided to
both `store.findRecord()` and `store.findAll()` inside the already existing
`options` object. The `query` object would be passed to your adapter and
serialized into the URL of the request that is made to fetch the resource from a
server.

## Inclusion of Related Resources

The JSON API spec allows clients to control the [inclusion of related
resources][fetching-includes] by providing an `include` query parameter when
requesting a resource.

Including related resources using `store.findRecord()`:

```javascript
// GET /articles/1?include=comments

const article = this.store.findRecord('article', 1, {
  query: {
    include: 'comments'
  }
});
```

Including related resources using `store.findAll()`:

```javascript
// GET /articles?include=comments

const articles = this.store.findAll('article', {
  query: {
    include: 'comments'
  }
});
```

## Sparse Fieldsets

The JSON API spec also allows clients to request that endpoints only return
specific fields in the response on a per-type basis by including a
`fields[TYPE]` query parameter.

Here is an example of fetching an article along with its author. In this example
we're able to specify that we only want the title and body of the article and only
the name of its author.

```javascript
// GET /articles/1?include=author&fields[articles]=title,body&fields[users]=name

const article = this.store.findRecord('article', 1, {
  query: {
    include: 'author',
    fields: {
      articles: 'title,body',
      users: 'name'
    }
  }
});
```

Similarly, this can also be done when requesting all articles:

```javascript
// GET /articles?include=author&fields[articles]=title,body&fields[users]=name

const articles = this.store.findAll('article', {
  query: {
    include: 'author',
    fields: {
      articles: 'title,body',
      users: 'name'
    }
  }
});
```

## This is JSON API Specific

No :simple_smile:. While these examples were for JSON API specific features,
there is nothing preventing users from passing whatever query parameters they
might need for features of their own API.

## Implementation

Since `store.findRecord()` and `store.findAll` already accept an `options`
object, their public APIs will not need to be changed. The private finders that
they use, however, will need to be modified to pass the `query` (or all of
`options`) through to their corresponding adapter methods. An example of one of
the places that would require change is
[here](https://github.com/emberjs/data/blob/8f83fbf27acdd642cb824e3286832acebdac0da1/packages/ember-data/lib/system/store/finders.js#L19).

Additionally, the find methods inside adapters will need to accept another
parameter. The changes could be something like:

```javascript
// Old
findRecord: function(store, type, id, snapshot) {
  return this.ajax(this.buildURL(type.modelName, id, snapshot, 'findRecord'), 'GET');
}

// New
findRecord: function(store, type, id, snapshot, query={}) {
  const url = this.buildURL(type.modelName, id, snapshot, 'findRecord');

  return this.ajax(url, 'GET', { data: query });
}
```

# Drawbacks

Why should we *not* do this?

- Adding a `query` object as an option to `store.findRecord()` and
  `store.findAll` makes knowing when to use `store.query` and
  `store.queryRecord()` less obvious by providing more than one way to do the
  some of the same things.
- This would add more surface area to the API, and require more maintenance/docs

# Alternatives

## What other designs have been considered?

We could continue using `store.query()` and `store.queryRecord()` whenever we
need to provide query parameters.

## What is the impact of not doing this?

It remains difficult to fully make use of all of the features provided by JSON
API or support features that other APIs might allow.

### Related Issues

- https://github.com/emberjs/data/issues/3596
- https://github.com/emberjs/data/issues/2905

# Unresolved questions

## Knowing When to Fetch

It's not totally obvious to me what impact this should have (if any) in terms of
when to fetch data from the server. We could always fetch data from the server
if any query parameters are passed but I think the better default is to rely on
loading the data in the background to make sure that we eventually end up with
all of the data that we need. If a user wants to make sure that they always have
fresh data, they can do so by setting the `reload` option to `true`.

[json-api]: http://jsonapi.org/ "JSON API Specification"
[fetching-includes]: http://jsonapi.org/format/#fetching-includes "JSON API - Inclusion of Related Resources"
[fetching-sparse-fieldsets]: http://jsonapi.org/format/#fetching-sparse-fieldsets "JSON API - Fetching Sparse Fieldsets"
