- Start Date: 2016-04-15
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

I am proposing to introduce a new API for manipulating query parameters being
sent by query() and queryRecord(). This will involve adding two new methods
to Adapters: a `buildQuery()` hook, and; a `transformParamKey()`
hook, to compliment the existing `sortQueryParams()`

# Motivation

When dealing with query parameters, the default behavior on the Adapter is simply
to alphabetize the keys in order.  So, what happens if you want to do more?

One common use case I (and others) have come across is the need to transform, or
"normalize" the keys from, for example, camelized to underscored. The current
solution is to override `sortQueryParams()`, but this hook name is not a semantic
description of what we really want to achieve. Furthermore, there is no simple way
of overriding the method in this way, without needing to reimplement the entire hook,
if you also wanted to preserve the default sorting behavior.

This RFC seeks to break the process down into a couple smaller pieces to make the
process overall process simpler.

# Detailed design

Currently the Adapter's `query()` and `queryRecord()` hooks call on
`sortQueryParams()`, which does its sorting and returns a new, sorted hash.

The proposed new API will instead call `buildQuery()`, which will delegate to
`sortQueryParams()` and the new `transformParamKey()` and return the resulting
hash.

Here is the example of how the new `buildQuery()` might look:

```js
buildQuery(query) {
  let sortedQuery = this.sortQueryParams(query);

  let sortedQueryKeys = Object.keys(sortedQuery);

  let finalQuery = sortedQueryKeys.reduce((object, key) => {
    let transformedKey = this.transformParamKey(key);
    return object[transformedKey] = sortedQuery[key];
  }, {});

  return finalQuery;
}
```

Here is the example of how the new `transformParamKey()` might look:

```js
  transformParamKey(key) {
    return key;
  }
```


# How We Teach This

The `buildQuery()` hook was written in a way that leaves the original
`sortQueryParams()` untouched, so in the end, a developer that needs to update
their code should only need to worry about replacing `this.sortQueryParams()` with
`this.buildQuery()` within their `query()` and `queryRecord()` methods.

`transformParamKey()` does nothing but return the key by default, so only an
API documentation should be sufficient to describe it's use.

# Drawbacks

One of drawbacks of this approach is that params must be sorted before the keys can
be transformed. Once the keys are transformed, it becomes trickier to reconstruct
the final param hash from the original hash.  This is another reason why I preserved
the original `sortQueryParams()` method.. as it returns a new hash that we can
further manipulate without any loss of detail.

This of course means that the new "sorted" hash needs to again be deconstructed into
keys and reduced finally into a finished hash.. this may not be the most efficient
thing to do.

# Alternatives

A reasonable alternative would be to simply _rename_ `sortQueryParams()` to `buildQuery()`.
Doing so makes it easier to just pop in the new `transformParamKey()` method in the loop
that builds the final param hash.

Example:

```js
sortQueryParams(obj) {
  let keys = Object.keys(obj);
  let len = keys.length;
  if (len < 2) {
    return obj;
  }
  let newQueryParams = {};
  let sortedKeys = keys.sort();

  for (let i = 0; i < len; i++) {
    let transformedKey = this.transformParamKey(sortedKeys[i]);
    newQueryParams[transformedKey] = obj[sortedKeys[i]];
  }
  return newQueryParams;
},
```
# Unresolved questions

Should this proposal go one step further and re-implement `sortQueryParams()`?
if the default behavior were changed to:

```js
sortQueryParams(arrayOfKeys) {
  return arrayOfKeys.sort();
}
```

this would simplify the overall implementation for `buildQuery()`:

```js
buildQuery(query) {
  let keys = Object.keys(query);

  let sortedKeys = this.sortQueryParams(keys);

  let finalQuery = sortedKeys.reduce((object, key) => {
    let transformedKey = this.transformParamKey(key);
    return object[transformedKey] = query[key];
  }, {});

  return finalQuery;
}
```

This of course would require a little more teaching and a little more change for
people that _do_ already have a `sortQueryParams()` implemented.
