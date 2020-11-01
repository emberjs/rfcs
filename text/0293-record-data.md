---
Start Date: 2018-01-10
Relevant Team(s): Ember Data
RFC PR: github.com/emberjs/rfcs/pull/293
Tracking: https://github.com/emberjs/rfc-tracking/issues/24

---

### Summary

Currently, incrementally experimenting with Ember Data internals is hard both for addon authors
and Ember Data contributors. This RFC rationalizes the internals and establishes clear boundaries
for record data storage and manipulation allowing us to expose a public api for addon authors to experiment with. 

### Motivation

Externally, addons can customize how apps communicate with the server by implementing the Adapter/Serializer APIs but changing how ED deals with relationships, attribute buckets, rollbacks, dirtyness
and similar issues is extremely challenging and impossible without extremely internal hacks. One can look at popular addons like EmberDataModelFragments
and see how many private APIs they had to override and hack to implement their funcionality.

Internally, while ED is reasonably well factored between data coming into the system through
Adapter/Serializers/IdentityMap/Store and data going out through DS.Model/Snapshots/Adapters/Serializers
, internal handling of the data including relationships and attributes has extremely fuzzy and unclear boundaries.

Data currently lives in internalModels, relationship state objects, computed property caches, relationship
payload caches, etc.

#### before
 ![image](https://user-images.githubusercontent.com/715175/33340994-6380c66a-d432-11e7-9f00-ed905e78915a.png)

This RFC proposes rationalizing and extracting ED's core record data handling layer into a RecordData class.

#### after
![image](https://user-images.githubusercontent.com/715175/33341155-e5f170c2-d432-11e7-9c50-4a3e977331fe.png)

This will allow us to rationalize internal ED APIs, establish clearer internal boundaries, 
allow experimentation by addon authors, and create a path for internal ED experimentation.

You can think of Record Data as a layer that can receive JSON api payloads for a record,
apply local changes to it, and can be queried for the current state of the data.

Examples of things this would enable:

1) By shipping a custom RecordData, EmberDataModelFragments can implement a large part of their 
funcionality without relying on private apis. Spike at [model fragments](https://github.com/igorT/ember-data.model-fragments/tree/igor/model-data)

2) A spike of Ember Data backed by Orbit, can be implemented as an addon, where most of the work
is in implementing a Record Data backed by Orbit. Spike at [data-orbit](https://github.com/igorT/data-orbit/tree/orbit-model-data)

3) By using an ES6 class for Record Data implementation, this brings us closer to an Emberless 
Ember Data running.

4) If you needed to implement a GraphQL like projection API, Adapters and Serializers would be enough
for the loading data, but currently there is no good place to handle client side data interactions.
RecordData would make it much easier to have a GraphQL ED addon

5) Certain apps and models have a large amount of read only data, which is currently very performance heavy
to implement in ED. They could use a read only fast record data addon, which would enable a large perf win.

6) Experimenting with schemaless approaches is currently very hard in ED, because internal
models encode assumptions of how attributes and relationships work. Having a swappable RecordData would
make it easier for us to implement schemaless approaches in addons.

7) By having Record Data fully expressed in JSON API apis, the current state of the store becomes serializable.

By designing a public interface for RecordData that dosen't rely on any other part of EDs current system,
we can use RecordData as the main building block around which we can refactor the rest of ED.


### Detailed design


#### High level design

Ember Data would define a RecordData interface, and ship a default implementation. Addons would
be able to swap their own implementation of the RecordData interface.

RecordData is an interface defining the api for how the store and DS.Models 
store and apply changes to data. RecordDatas hold
the backing data for each record, and act as a bridge between the Store, DS.Model, and Snapshots.
 It is per record, and defines apis that respond to 
store api calls like `pushData`, `adapterDidCommit` and DS.Model updates like `setAttribute`.
RecordData represents the bucket of state that is backing a particular DS.Model.

The store instantiates the RecordData, feeds it JSON API data coming from the server and
tells it about state changes. DS.Model queries the RecordData for the attribute
and relationship values and sends back the updates the user has made.

Other than the `storeApisWrapper` passed to it, RecordData does not assume existence of
any other Ember or Ember Data object. It is a fully self contained system, that might serve
as a basic building block of non Ember/ED data libraries and could be extracted into a separate
library.

#### Interface

The interface for RecordData is:

```js
export default class RecordData {
  constructor(modelName: string, clientId?: string, id?: string, storeApisWrapper: StoreApisWrapper) {
    /*
      Exposing the entire store api to the RecordData seems very risky and would 
      limit the kind of refactors we can do in the future. We would provide a wrapper
      to the RecordData that would enable funcionality MD absolutely needs 
    */
  }


  /*
    Hooks through which the store tells the Record Data about the data
    changes. They all take JSON API and return a list of keys that the 
    record will need to update
  */

  pushData(data: JsonApi, shouldCalculateChanges: boolean/* if false, don't need to return changed keys*/) {
  }

  adapterDidCommit(data: JsonApi) {
  }

  didCreateLocally(properties) {
  }

  /*
    Hooks through which the store tells RecordData about the lifecycle of the data,
    allowing it to keep track of dirtyness
  */

  adapterWillCommit(modelName: string, id?: string, clientId?: string) {
  }

  saveWasRejected(modelName: string, id?: string, clientId?: string) {
  }

  adapterDidDelete(modelName: string, id?: string, clientId?: string) {
  }

  recordUnloaded(modelName: string, id?: string, clientId?: string) {
  }


  /*
   Rollback handling
  */

  rollbackAttributes(modelName: string, id?: string, clientId?: string) {
  }

  rollbackAttribute(modelName: string, id?: string, clientId?: string, attribute: string) {
  }

  changedAttributes(modelName: string, id?: string, clientId?: string) {
  }

  hasChangedAttributes(modelName: string, id?: string, clientId?: string) {
  }


  /*
    Methods through which DS.Model interacts with RecordData, by setting and getting local state
  */

  setAttr(modelName: string, id?: string, clientId?: string, key: string, value: string) {
  }

  getAttr(modelName: string, id?: string, clientId?: string, key: string) {
  }

  hasAttr(modelName: string, id?: string, clientId?: string, key: string) {
  }

  /*
    Relationships take and return json api resource objects
    The store takes those references and decides whether it needs to load them, or
    it can serve them from the cache
  */

  getHasMany(modelName: string, id?: string, clientId?: string, key: string) {
  }
  
  addToHasMany(modelName: string, id?: string, clientId?: string, key: string, jsonApiResources, idx: number) {
  }

  removeFromHasMany(modelName: string, id?: string, clientId?: string, key: string, jsonApiResources) {
  }

  setHasMany(modelName: string, id?: string, clientId?: string, key: string, jsonApiResources) {
  }

  getBelongsTo(modelName: string, id?: string, clientId?: string, key: string) {
  }

  setBelongsTo(modelName: string, id?: string, clientId?: string, key: string, jsonApiResource) {
  }

```


```js

export default class StoreApiWrapper {
  /* clientId is used as a fallback in the case of client side creation */
  createRecordDataFor(modelName, id, clientId)
  notifyPropertyChanges(modelName, id, clientId, keys)
  /* 
  in order to not expose ModelClasses to RecordData, we need to supply it with
  model schema information. Because a schema design is out of scope for this RFC,
  for now we expose these two methods we intend to deprecate once we have a schema
  interpretation
   */
  attributesDefinitionFor(modelName, id)
  relationshipsDefinitionFor(modelName, id)

}
```


#### ED's usage of RecordData
We would refactor internal models, DS.Models and Snapshots to use RecordData's apis.

Reimplementation of ED current internals on top of RecordData apis would consist of the store
pushing the json api payload to the backing record data and the record data setting up internal
data tracking, as well as storing relationship data on any additional needed recordDatas.


```js
let data = {
  data: {
    id:1,
    type: 'user',
    attributes: { name: 'Clemens' },
    relationships: { houses: { data: [{ id: 5, type: 'house' }], links: { related: '/houses' } } }
  }
};

store.push(data);

// internal store method
_internalMethod() {
  let recordData = store.recordDataFor('user', 1, this._storeWrapperApi)
  recordData.pushData(data, false)
}

->

// model-data.js
pushData(data, shouldCalculateChanges) {
  this._data = this.data.attributes;
  this._setupRelationships(data);
}
->
// model-data.js
_setupRelationships(data) {
  this.storeWrapperApi.recordDataFor('house', 1);
  ....
}
```
   
The DS.Model interactions would look like:

```js
let user = store.peekRecord('user', 1);
user.get('name');
->
// DS.Model
get(key) {
  let recordData = _internalMethodForGettingTheCorrespondingRecordData(this);
  return recordData.getAttr('name');
}
```

#### Relationships

##### Basic loading of relationships

RecordData's relationship hooks would receive and return json api relationship objects with
additional metadata meaningful to Ember Data.

Lets say that we started off with the same user data as above

```js
let data = {
  data: {
    id:1,
    type: 'user',
    attributes: { name: 'Clemens' },
    relationships: { houses: { data: [{ id: 5, type: 'house' }], links: { related: '/houses' } } }
  }
};
let clemens = store.push(data);
```

Getting a relationships from Clemens would trace a path from the DS.Model to backing record data,
which would then give the store a json api object, and the store would instantiate a ManyArray
with the records


```js
clemens.get('houses');
// DS.Model
get() {
  let clemensRecordData = _internalApiGetsUsTheRecordDataFromIDMMAP();
  return clemens.getHasMany('houses');
}
->
// Record Data returns
{[ 
  data: { id: 5, type: 'house'},
  links: { related: '/houses' },
  meta: { realMetaFromServer: 'hi', _ED: { hasAllIds: true, needToLoadLink: false } }
}
-> //store takes the above, figures out that it needs to fetch house with id 5
  // and returns a promise which resolves into a ManyArray

```

ED extends the relationship payload with a custom meta, which gives the store information 
about whether we have information about the entire relationship (we couldn't be sure we
have all the ids if we loaded from the belongsTo side) and whether the link should be refetched
(we might need to refetch the link in the case it potentially changed)


##### Setting relationship data locally

Similarly to the attributes, changing relationships locally tells record data to update
the backing data store
```js
let anotherHouse = store.push({data: { type: 'house', id: '5' }});
clemens.get('houses').then((houses) => {
  houses.pushObject(anotherHouse);
  -> 
  // internally
  clemensRecordData.addToHasMany('houses', { data: { type: 'house', id: '5' } })
});
```


##### Dealing with newly created records in relationships

Unfortunately, because ED does not have first class clientId support, we need a special case
for handling locally created records, and pushing them to relationships.

We extend JSON API resource object with a `clientId` meta field.
A locally created record, will also have a ED specific internal client id, which will take preference;

```js
let newHouse = store.createRecord('house');
clemens.get('houses').then((houses) => {
  houses.pushObject(newHouse);
  ->
  // internally
  clemensRecordData.addToHasMany('houses', { data: { type: 'house', id: null, { meta: _ED: { clientId: 1}} } })
});
clemens.get('houses') ->
{ data: 
  [ { id: 5, type: 'house'}, 
    { id: null, type: 'house', meta: { _ED: { clientId: 1 } } }],
  links: { related: '/hi' },
  meta: { realMetaFromServer: 'hi', _ED: { loaded: true, needToLoadLink: false } }
}
```


ED internals would keep a separate cache of client ID and resolve the correct record


#### Addon usage

The Store provides a public api for looking up a recordData which the store has not seen before.

```
recordDataFor(modelName, id, options) {

}
```

If an Addon wanted to implement custom data handling functionality, it would subclass the store
and implement their own RecordData handler.

There are three main reasons to do this.

1. Full replacement of Ember Data's data handling mechanisms

Best example would be the Ember Data backed by Orbit.js experiment. EmberDataOrbit Addon replaces
Ember Data's backing data implementation with Orbit.js. Most of this work can be done by EmberDataOrbit
replacing ED's Record Data implementation

```
recordDataFor(modelName, id, options, storeWrapper) {
  return new OrbitRecordData(modelName, id, storeApisWrapper) 
}
```

2. Per Model replacement of Ember Data's data handling

If a large app was loading thousands of instances of a particular record type, which was read-only,
it could use a read only ED addon, which implemented a simplified RecordData without any change tracking.

The addon would implement a `recordDataFor` on the store as 

```
recordDataFor(modelName, id, options, storeWrapper) {
  if (addonDecidesIfReadOnly(modelName))  {
    return new ReadOnlyRecordData(modelName, id, storeApisWrapper) 
  }
  return this._super(modelName, id, options, storeWrapper);
}
```

3. Adding common funcionality to all ED models

Ember Data Model Fragments Addon adds support for handling of embedded data fragments.
In order to manage the handling of fragments, Model Fragments would compose ED's default
RecordData with it's own for handling fragments.


```js
recordDataFor(modelName, id, options, storeWrapper) {
  let EDRecordData = this._super(modelName, id, options, storeWrapper);
  return new ModelFragmentsRecordData(modelName, id, options, storeWrapper, EDRecordData);
}
```

When receiving a payload, ModelFragments would handle the fragment part and delegate the rest
to ED's implementation

```js
pushData(data, shouldCalculateChanges) {
  let keysThatChanged = this.extractAndHandleFragments(data);
  return keysThatChanged.concat(this.EDRecordData.pushData(data, shouldCalculateChanges))
}
```

### How we teach this

These APIs are not meant to be used by most users, or app level code, and should be hidden away and
described in an api/guides section meant for ED addon authors. Currently there are a few widely used
addons which would greatly benefit from this, so we can also reach out in person. I have already implemented
a spike of ModelFragments using RecordData. Having couple addons implement different RecordDatas would be
a great way to teach new addon authors about the purpose and implementation of the API.

### Drawbacks

#### Defines a bigger API surface area

This change would increase the public API surface area, in a codebase that is already pretty complex.
However, this would codify and simplifyA APIs addon authors have already had to interact with, while
creating a path for future simplification of the codebase. 

#### It allows people to do very non-standard changes that will complexify their app needlessly

The main mitigation, is only giving RecordData access to a small amount of knowledge of the external world,
and keeping most APIs pull only thus discouraging trying to do innapropriate work in the RecordData layer

#### The new JSON api interaction might preclude performance improvements, or reduce current performance


### Alternatives

#### We could do this work as an internal refactor, and not expose it to public.

I believe that this approach is valid as an internal architecture, so would like to do it even if
we did not expose any of it to addons/apps.

#### Make RecordData's looked up from the resolver

Currently RecordData is a dumb ES6 class and does not live in the Ember resolver system, for performance
and simplicity reasons. We could alternatively look it up from the resolver, allowing people
to mock it and inject into it easier.

#### Don't expect a per record Record Data

Currently, the MD layer semantics mimics current ED's data storage, where data is stored per record in
internalModels. You could alternatively do this using an app wide cache, like Orbit.js does, or
using any number of other approaches. This approach while valid, would be harder to implement and
it's apis would not map as well to ED behavior.

### Open Questions

#### Versioning and stability

Our current implementation of `internalModel` is deeply monkeypatched by at least few addons. I think
we have to consider it as an semi-intimate api, even though it literally has `internal` in the name(I've been told adding couple undescores to the name would have helped).
Because the number of addons monkeypatching it is limited, we can manually migrate them onto the new
apis. However this requires us to make the new apis public from the get go, and doesn't allow for a long period of api evolution. 

The following options are available, none of them great:

1) Feature flag RecordData work. The scope of this refactor is large enough, that doing a full feature
flagging would be an enourmous burden to bear, and I would advise against it. We can proxy some basic
things, to allow for simpler changes and as a way of warning/deprecating

2) Move from the internals to public RecordData in a single release cycle, and hope public apis we created
make sense, and will not be performance issues in the future. I am reasonably confident having implemented
several addons using RecordData that the basic design works, but things can always come up.

3) Move from private internals to private RecordData, and then feature flag the public apis over couple
versions. In this case the addons monkeypatching the internals, would monkeypatch the new nicer apis
for a while, and then easily switch to the public api. This feel a bit like SemVer cheating.

#### ClientID passing to store api methods

We use `recordDataFor(modelName, id, clientId)` as the api to look up recordDatas. Passing an often
null clientId seems annoying. Orbit.js uses an identity object instead, and if we did the api would look like `recordDataFor(identityObject)`, where `identityObject` would look like `{ type, id, meta: { _ED: { clientId }}}`. This seem a bit more correct, but doesn't look like any existing ED api, and could create
a lot of allocations.

#### RecordDatas might need to do some global setup/communication, how does that work?

Normally you would do this in an initializer, but becasue MDs aren't resolved, the only way would be
to do it in RecordDataFor or by using a singleton import. Some ceremony being required to using RecordData
isn't super bad, because it will discourage app authors from customizing it for trivial/innapropriate
things.

#### What do we do with the record state management? 

Currently RecordData has no interaction with the state machine. I think we should punt on this
for now. 

#### { meta: { _ED: { props here } } } alternatives?

We could put the ED internal data outside of meta, and keep meta only for actual meta that comes from
the server.

#### Naming of everything

Please help with better names for things if you have ideas

#### Snapshot interface

How does a Snapshot ask Record Data for it's attributes

#### Real life perf impact

Need benchmarks
