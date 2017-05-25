- Start Date: 2014-08-22
- RFC PR: 
- Ember Issue: 

# Summary

Extend Ember data Model prototypes to have individual 'updatable' properties. Model instances can receive patch operations in response to updating or deleting records (PUT or DELETE requests) by pushing **partial** changes into the store (DS.Store#update). Create mixins for Model, Adapter & Serializer prototypes which support using a JSON Patch [rfc6902] format in payloads sent as responses in order to apply partial updates.

[rfc6902]: http://tools.ietf.org/html/rfc6902

# Motivation

A solution is needed to minimize the occurance creating a stale cache in memory as many records change. And, to prevent clobbering current changes (in progress) while editing a complex web form a solution is needed to respond to related changes in a record or in many changed records. 

Data relationships that persist foreign keys, e.g. (`related_ids` or `related_id`) often need to be updated after the related record(s) change. 

Related records may contain cached properties to prevent extraneous HTTP requests. As related records are updated their associated records may need an update as well. 

Related records may be saved in a batch ([bucket]) process. Aside from the batch, new related records can be created. Many records in a complex web form can be persisted asynchronously.

A workaround that is used is manually reloading related records after completing an update (PUT request).

## Use Cases:

* When a (parent) record is edited using a web form, and a (related) record is added which requires the parent to be updated (patched or reloaded), an undesirable result may be that the parent record's attributes are overwritten.

* When a related record is deleted a parent record may require that the foreign key of deleted record be removed from it's list of relations (record array). Thus a parent record needs to receive an update (perhaps a patch operation). Likewise, when a new record is added, it's id needs to be tracked.

* When records are related using a "has and belongs to many" (HABTM) association, foreign keys are stored on both of the related records. A list of related records may be used as the sort order for displaying the associated records by rendering a template with an `each` iterator. When a related record is created or removed the associated record(s) requires an update to the list of associations.

* When records are related with an 'async' option; in order to serialize the record with an acurate list of related ids, every related record's promise must be resolved first. Instead of using Ember Data relationships, perhaps using computed properties would provide synchronous access to the list of related ids when serializing.

* When the RESTAdapter makes a PUT request and the server returns a payload the serializer's extractSingle or extractArray which normally expects the complete payload, not a partial, see [DS.RESTAdapter.html#ajax]. Perhaps a 206 (partial) repsonse can be supported using [DS.Store.html#update] to update the related records.

Each one of these cases may involve creating, updating or deleting using RESTful requests. It may be possible for a server to return a payload that has changes to be applied to the application's store (distributed cache, in memory). Some of the cases above may include implementations on both the client and server. Perhaps server responses can be tuned to supply the necessary changes after PUT and DELETE request complete.

[bucket]: https://github.com/pixelhandler/ember-bucket
[Ember.DAG]: https://github.com/emberjs/ember.js/blob/master/packages/ember-application/lib/system/dag.js
[Directed acyclic graph]: https://en.wikipedia.org/wiki/Directed_acyclic_graph
[Ember.Application#initializer]: http://emberjs.com/api/classes/Ember.Application.html#method_initializer
[DS.Store.html#update]: http://emberjs.com/api/data/classes/DS.Store.html#method_update
[DS.RESTAdapter.html#ajax]: http://emberjs.com/api/data/classes/DS.RESTAdapter.html#method_ajax


# Detailed design

Create functionality that supports [JSON Patch] operations. Using the JSON Patch specification, [rfc6902], for change operations, a model can manage updates on a per property basis using (HTTP) PATCH requests. 

Instead of sending a request with a complete document, a patch operation can be used to persist individual property changes. Upon success of the patch operation, which includes a test to verify the record is not stale, the record may transition to a clean state.  The model's state machine may be augmented to add isPatching[Property] sub-states that can be used to disable web form inputs or other action buttons; or perhaps using flags for each field that represent the state of that field, active or disabled.

```json
[
  { "op": "replace", "path": "/resource/id/property", "value": 42 },
  { "op": "test", "path": "/resource/id/property", "value": 0 }
]
```

[JSON Patch]: http://jsonpatch.com
[rfc6902]: http://tools.ietf.org/html/rfc6902

## Technical Considerations

* Mixin for (Ember Data) DS.Adapter/DS.Serializer for PATCH requests
* Mixin for DS.Model to support changes that fire PATCH requests
  * Use fake computed properties for the form fields that fire change on blur events
  * Use flags or states to indicate the `active` or `disabled` status for properties

## Acceptance Tests

Given a Web form representing an existing record  

When a change to a form field is triggered by exiting the field  
Then a REST adapter handles a PATCH request using a replace operation  
  And while the field is patching it is disabled (editing prevented)

When a change is made that results in updates to related records  
Then the server responds with the necessary operation to update the related records  
  And the client application patches the records, users may notice the update or not

Given a change set, operation(s) sent in response to an update or delete (HTTP) request

When the client application has the related record(s) in its store (memory cache) needing an operation  
Then the store updates its cache according to the operations

When the client application does not have the related record(s) in its store (memory cache) needing an operation  
Then the store's adapter makes a request for the records needing an operation

# Drawbacks

If there is a better proposal, aside from JSON Patch, another solution for partial updates should be considered. 

[JSON API] does include JSON Patch, "A JSON API server may also optionally support modification of resources with the HTTP PATCH method [RFC5789] and the JSON Patch format [RFC6902]."

[JSON API]: http://jsonapi.org/format/
[RFC6902]: http://tools.ietf.org/html/rfc6902
[RFC5789]: http://tools.ietf.org/html/rfc5789

# Alternatives

One solution to the problem of saving many records (complete representations of a resource)asynchronously and enforcing an order of operations, may be using a strategy like [Ember.DAG] provides, a [Directed acyclic graph] used by the [Ember.Application#initializer] method to order the instantiation of objects for initialization (uses `before` and `after` settings for each object that is initialized).

Another solution may be to introduce code from [Orbit.js] specifically the requestable and transformable interfaces.

[Orbit.js]: https://github.com/orbitjs/orbit.js

# Unresolved questions

1. Explore responses to PUT and DELETE requests, then update the records with the payload contents, based on response status 200 (complete representation) or status 226 (partial representation of the resource).

1. Explore using JSON Patch specification for payloads using HTTP PATCH requests to update properties as fields change (after user exits the field).

1. Explore using JSON Patch spec for payloads sent with a 226 status, then update the records as the patch operations indicate.

1. Is a graph object needed which provides strategy for processing patch operations. If all models can patch at the property level then it may not be necessary to enforce an order of operations.