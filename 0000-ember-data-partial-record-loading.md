- Start Date: 2015-01-03
- RFC PR:
- Ember Issue:

# Summary

Partial record loading is a method of loading incomplete representation of the
record and fetching the rest of the data when it's needed.

# Motivation

Partial record loading is useful for APIs that may return an incomplete
representation of the record (for example due to performance optimisation).
Let's take a `repository` resource which returns `last_build` as a part of a
response:

```json
{
  "repository": {
    "id": 1,
    "owner": "emberjs",
    "name": "ember.js",
    "last_build": {
      "type": "build",
      "id": 999,
      "state": "passed",
      "number": 333
    }
  }
}
```

The `last_build` hash is an incomplete representation of a record of a `build`
type. It can be beneficial to load such a record to the store right away and
load the rest of the data when it's accessed.

For example:

```javascript
store.push('build', repository['last_build'])
var build = store.find('build', repository['last_build']['id'])

build.get('startedAt'); // will trigger an ajax query

build.get('isUpdating'); //=> true (could be also isLoading)

// eventually
build.get('startedAt'); //=> '2014-01-01T10:22:33'
```

# Detailed design

The simplest algorithm for handling partial records is to run an ajax query to
fetch a record whenever a property that is not defined is accessed. Given a
following record definition:

```javascript
var Post = DS.Model.extend({
  title: DS.attr(),
  content: DS.attr()
});
```

a partial record could be a post with just a title, for example loaded from
pusher as follows:

```javascript
store.push('post', { id: 1, title: 'Partial record loading in EmberData' });
```

Accessing any property that's not present in a payload, in this case `content`,
should trigger an update.

This should handle most of the use cases. For all of the other use cases there
should be a method to override the default behaviour.

Cases, which are not covered by a "happy path", are:

1. A property is not returned by the API. Some APIs may ommit some properties if
   they're not needed for a given record. This could result in triggering
   multiple requests to refresh the model. To handle this case we would need to
   allow specyfing fields that can be potentially missing. We could also add a
   fallback - if a field is not available even after loading a "full"
   representation we could mark it to not try to fetch it again.

2. A property may not be needed at a given point and should not trigger fetching
   the full representation. For example given a `Build` model which have a
   `startedAt` and `finishedAt` properties, there's no need to fetch the
   `finishedAt` for builds that are still running. There should be a way to
    specify a conditions for a given field.

3. Saving. If some of the properties are missing, we can't serialize a full
   record, because missing properties would be overwritten. I think that the
   default behaviour should be to raise an error if such a situation occurs,
   that is: someone wants to save an incomplete record. That said, it should be
   possible to customize this and send only loaded properties.

# Drawbacks

Partial records loading may mean different things to different people. Although
it doesn't seem complicated for simple cases, it could get messy to maintain.
It may be better to leave it as an addon, but then it would be great for Ember
Data to provide more hooks to plug into.

# Alternatives

This feature can be implemented manually, but currently it needs an internal
API.

# Unresolved questions

This proposal is a bit vague in its current form, but I hope to refine it if the
base idea is approved.
