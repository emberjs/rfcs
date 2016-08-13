- Start Date: 2016-10-11
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Add public hooks on `rest`-adapter which can be used to customize the
properties of an AJAX request being made for an adapter operation.

# Motivation

Currently `host`, `namespace`, `urlForXXX()` and `headers` is the only public
API on the `rest`-adapter which allow you to customize the request for an
adapter operation like `findRecord`, `createRecord`, .... Technically `ajax`
and `ajaxOptions` are private, but they have been overwritten in apps and
addons due to the lack of a public API.

The aim of the `ds-improved-ajax` feature implemented in
[#3099](https://github.com/emberjs/data/pull/3099) was to address the lack of
public API for customizing a request. Unfortunately, it hasn't gone through a
proper RFC process :pensive:. The feature is not enabled and available in a
`release`, so this RFC should flesh out the specific API for the feature before
it can be enabled eventually.

The result of this RFC should make sure that the new hooks allow developers to
customize a request with only public API.

# Detailed design

An adapter operation is split up into 2 steps

- 1) collecting the properties for the request, i.e. `_requestFor`
- 2) executing the request, i.e. `_makeRequest`

So for a `findRecord` it would look like this:

``` js
findRecord(store, type, id, snapshot) {
  let requestOptions = this._requestFor({
    store, type, id, snapshot,
    requestType: "findRecord"
  });

  return RSVP.resolve(requestOptions).then(this._makeRequest);
}
```
### `_requestFor`

Each hook for step 1) is invoked with all the arguments which are passed to the
corresponding adapter method. Additionally `requestType` is set so the hooks
know for which specific adapter operation they are invoked. To allow for
asynchronous configurations (e.g. pre-flight request to refresh an
authentication token), `_requestFor` returns a promise which resolves with an
object describing the request to make.

``` js
_requestFor(params) {
  let method  = this.methodForRequest(params);
  let url     = this.urlForRequest(params);
  let headers = this.headersForRequest(params);
  let body    = this.bodyForRequest(params);

  return RSVP.hash({ method, url, headers, body });
}
```
##### `methodForRequest`

Invoked to get the HTTP method for the request. Should return one of `"GET"`,
`"PUT"`, `"POST"` or `"DELETE"`.

##### `urlForRequest`

Get the URL for the request. This hook returns the URL for the request which
should be made, e.g. `/users/1` or `api.example.com:4321/users?name=abc`

This hook uses 2 new sub-hooks to construct the URL:

- `pathForRequest` - re-uses the existing `urlForXXX` hooks
- `queryForRequest` - get an object for all query params for the request

The existing [`sortQueryParams`](http://emberjs.com/api/data/classes/DS.RESTAdapter.html#method_sortQueryParams)
hook is reused within the new `queryForRequest`. An warning/deprecation should
be logged when the path already contains query params, as the new
`queryForRequest` hook should be used for that.

Pseudo implementation:

``` js
urlForRequest(options) {
  let path = this.pathForRequest(options);
  let query = this.queryForRequest(options);

  if (path.indexOf('?') !== -1) {
    warning("use queryForRequest to specify query params");
  } else if (query) {
    let sortedQueryParams = this.sortQueryParams(query);
    let queryParams = Ember.$.param(sortedQueryParams);

    return `${path}?${queryParams}`;
  }

  return path;
},

pathForRequest(options) {
  // re-use existing urlForXXX hooks
},

queryForRequest({ requestType, query }) {
  if (requestType === "query") {
    return query;
  }
}
```
##### `bodyForRequest` (currently `dataForRequest` in `ds-improved-ajax`)

Body for the HTTP request which should be made.

**Note**: in the current implementation of `ds-improved-ajax` the
`bodyForRequest` hook is called `dataForRequest`. To be more agnostic of the
underlying library used for making the request, it has been renamed to
`bodyForRequest` within this RFC.

##### `headersForRequest`

All HTTP headers for this request. By default it returns the value for the
`headers` computed property which can be set on the adapter. This hooks allows
to add custom headers for each request individually.

### `_makeRequest`

The `_makeRequest` is invoked with all the properties that describe the request
which should be made.

``` js
_makeRequest(request) {
  // convert request to jQuery hash
  // invoke Ember.$.ajax and make it runloop aware
  // return Promise which resolves/rejects with made request
}
```

# How We Teach This

- dedicated section in the blog post, since this feature has been in `beta`
  since `2.6`
- dedicated section in the guides [on customizing adapters](https://guides.emberjs.com/v2.8.0/models/customizing-adapters/)

# Drawbacks

- increased API surface
- current proposal of promisified hooks make overwriting using `_super` cumbersome:

``` js
headersForRequest(options) {
  let _headers = this._super(...arguments);

  return RSVP.resolve(_headers).then((headers) => {
    headers["X-Custom-Header"] = "my-custom-header";
    return headers;
  });
}
```
# Alternatives

- make `ajax` / `ajaxOptions` public
- make smaller API footprint, not so much hooks

# Unresolved questions

- the currently propose API doesn't allow to only use public API to
  [customize](https://github.com/emberjs/data/blob/2926c47453d50d6b75590d2ff447a4d0da66833a/addon/adapters/json-api.js#L226-L234)
  the `json-api`-adapter, though the current implementation could be changed to
  set the `Content-Type` header instead (which would be more agnostic anyway)
- currently `requestType` is used to indicate the specific adapter operation,
  but that name might not be to expressive. Should this be renamed to the more
  verbose, like `adapterOperation`?
- should `queryForRequest` be named `queryParamsForRequest`?
- should there be more hooks within `urlForRequest`, as outlined in the
  [Syntax](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier#Syntax)
  section on the URLs wiki page?
- what hooks are missing?
    - setting the `dataType` [#4357](https://github.com/emberjs/data/pull/4357)?
