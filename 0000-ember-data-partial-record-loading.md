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

The basic idea of the implementation of partial record loading is to:

* save a list of loaded attributes when a record is populated
* mark the record as partially loaded or fully loaded
* whenever one of the properties that are missing is accessed, a full record
  reload should be done
* if a server responded with all of the fields, the record should be marked as
  fully loaded

## Public API

This feature shouldn't be enabled by default, because it wouldn't be backwards
compatible and what's more, not every app needs it.
As long as an API returns all of the properties it shouldn't cause much problems,
but some APIs may for some reason omit an attribute instead of sending it
with a null value. Such a behvaviour would cause a record to always be
marked as partially loaded.

Partial loading support could be enabled in a serializer, by setting the
`enablePartialLoading` property:

```javascript
var Serializer = Ember.RESTSerializer.extend({
  enablePartialLoading: true
});
```

Setting this property turns on the default implementation.
If an application is read only a user doesn't have to
do anything else. If a server returns partial data, record is marked as
partially loaded and as soon as one of the missing properties is accessed the
record is refreshed by sending a query for the full record.

### Omitting properties

In some cases an API may omit one of the properties, even if the intent is not
to return a partial representation. Imagine an API that returns a key-value
pairs with an additional `public` field that informs if the value is private or
public. An example payload could like that:

```json
{
  "settings": [
    {
      "key": "username",
      "value": "drogus",
      "public": true
    },
    {
      "key": "password",
      "public": false
    }
  ]
```

The second element lacks a value property, because the API doesn't expose a
value of a private key-value pair.

In order to handle such case properly with partially loaded records enabled, a user
can add missing fields in the `normalize` method:

```javascript
var Serializer = Ember.RESTSerializer.extend({
  normalize: function(type, hash, prop) {
    if(type.typeKey === 'setting') {
      if(!hash[prop].hasOwnProperty('value')) {
        hash[prop].value = null;
      }
    }

    return hash;
  }
});
```

This is, however, not a very elegant way to do it. The intent of this code is
not clear. In order to make it simpler, we could introduce a simpler method for
this:

```javascript
var Serializer = Ember.RESTSerializer.extend({
  attr: {
    value: { partialLoadingCheck: false }
  }
});
```

Obviously it needs a better name, but something along the lines would make this
task very simple.

### Special logic for a partial load

A property may not be needed at a given point and should not trigger fetching
the full representation. For example given a `Build` model which have a
`startedAt` and `finishedAt` properties, there's no need to fetch the
`finishedAt` for builds that are still running.

In order to customise the default behaviour a new method on the Model class
should be introduced: `loadFullRecord`. Overriding it should allow to avoid
reloading if needed:

```javascript
var Build = DS.Model.extend({
  startedAt: DS.attr(),
  finishedAt: DS.attr(),
  state: DS.attr,

  loadFullRecord: function() {
    if(this.get('state') === 'running') {
      return false;
    } else {
      return this._super.apply(this, arguments);
    }
  }
});
```

### Special logic for adapter

An API may default to return a partial record and need a special query for a
full record. In such situation there needs to be an easy way to override a
default adapter behaviour. A new method should be introduced, similarly to a
record case: `loadFullRecord`. The default implementation would do a regular
find. An overrided version could look like so:

```javascript
var Adapter = DS.Adapter.extend({
  loadFullRecord: function(store, type, id, record) {
    var data = { full: true };
    return this.ajax(this.buildURL(type.typeKey, id, record), 'GET', data);
  }
});
```

### Changing properties on a partially loaded records

Changing properties on a partially loaded record may lead to problems, so we
should throw an error in such case. There should be, however, a way to do it, because
in some situations it may make sense. I'm not yet sure where should the API for
this case lay. On the one hand it would make sense to put it in a serializer,
just like it's enabled, but on the other hand the check will be probably made
in the model.

## Implementation

TODO

# Drawbacks

Partial records loading may mean different things to different people. Although
it doesn't seem complicated for simple cases, it could get messy to maintain.
It may be better to leave it as an addon, but then it would be great for Ember
Data to provide more hooks to plug into.

# Alternatives

This feature can be implemented manually, but currently it needs an internal
API.
