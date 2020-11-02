---
Start Date: 2018-11-25
Relevant Team(s): Ember Data
RFC PR: https://github.com/emberjs/rfcs/pull/403
Tracking: https://github.com/emberjs/rfc-tracking/issues/31

---

# Ember Data | Identifiers 

## Summary

`Identifiers` provides infrastructure for handling `identity` within `ember-data` to
 satisfy requirements around improved caching, serializability, replication, and handling
 of remote data.

This concept would parallel a similar structure proposed for `json-api` resource identifier
 `lid` property [drafted for version `1.2` of the `json-api` spec](https://github.com/json-api/json-api/pull/1244).

In doing so we provide a framework for future RFCs and/or addons to address many common
 feature requests.

----

## Motivation

This groundwork RFC represents the union of a diverse set of motivations, each of which 
 is discussed below in no particular order of importance, outside of the first. **This 
 RFC is not seeking to immediately address each of the motivations below, we are adding
 infrastructure to make future RFCs possible in these spaces**

### Unified concept of Identity

Identity is a core concept to managing a cache and guaranteeing [atomicity, consistency,
 isolation, and durability](https://en.wikipedia.org/wiki/ACID_(computer_science)). 
 Currently, `ember-data` has no unified mechanism for Identity. This missing mechanism 
 introduces errors in application code, makes `ember-data` internals needlessly complex, 
 and complicates the method signatures of many public APIs.

Creating a unified `Identity` concept will allow `ember-data` to expose `identity` as a 
first-class primitive to our users, improving the mental model, improving application 
resiliance to errors, and providing a clear language of communication between `ember-data`'s 
various primitives.

Today, we handle `identity` in a myriad of ad-hoc ways:

* `InternalModel` instances as keys for most internal methods and the relationship layer.
* `type+id` for serializing/deserializing a `record` or [`resource`](https://jsonapi.org/format/#document-resource-object-identification)
* `type+clientId` for caching newly created records on the client and communicating 
   about them with `RecordData`.
* Various other non-serializable forms of identity for tracking requests, relationship 
   membership and state.
* In some cases we have no concept at all where one is needed (for instance, caching 
  queries)

We wish to simplify and codify our handling of identity.

### Simplify StoreWrapper and RecordData APIs

Today, to deal with the lack of a unified `identifier` concept, we overload many 
 `StoreWrapper` and `RecordData` API method signatures with `modelName`, `id`, `clientId`
 as arguments. This leads to method signatures being long and needlessly unwieldy.

Note: We had initially intended to overload all of these classes method signatures in 
this way, to ensure that `RecordData` implementations could be `singleton`s, but we failed
to correctly implement the `RecordData` RFC in this regard and a near future RFC will look
at rectifying this using the `Identity` APIs introduced here.

Moving to a unified concept of `identity` opens up a path to clean up these method
 signatures.

### Operations

Operations are a foundational concept for `acid` transactions. Without the ability to 
 describe an operation and a clear mental model of what operations exist and achieve, it
 is difficult to understand how an action affects state. Granularity and clarity is key.

While `ember-data` does not yet have concepts of operations or transactions, mutations and
 updates are applied directly, there are many areas we could improve upon by introducing
them. As with data, operations upon data should be serializable so that local state can 
be accurately cached on local clients.

### Nested Saves / API Transactions / Websocket Support

Many applications wish to create or update and save multiple records together. Achieving
 this in `ember-data` today is unweildy and has many difficult edge cases: one of which
 is correctly matching data received back from the API to the newly created records already
 on the client.

A similar edge case occurs when a newly created record is saved for the first time and 
 prior to receiving the request response the same record is recieved via another means
(background polling, websocket subscription, etc.). We have no means of matching the record
returned by the alternative means to that of the request, leading to a second cache entry 
being created and an error once the initial request completes.

One solution has been to generate and assign `id`s for records on the client, but this is
 not always desireable. These scenarios are a major motivation for `lid` in the `json-api`
spec. Users wishing to solve these cases would be able to serialize the `lid` of the 
`Identifier` for a newly created record and reflect that `lid` back in any payloads send 
from their API for the session to correctly match the payloads to the record.

### Better Cache Serialization & Improved Infra for Offline Support

In order to enable users to achieve full offline support, or to serialize the store 
 or transport across the wire (for example as an advanced fastboot rehydration mode)
the entire state of the store needs to be serializable. This RFC introduces the 
foundation for mechanisms through which this can be later achieved.

----

## Detailed design

```typescript
export interface Identifier {
  lid: string;
}

export interface RecordIdentifier extends Identifier {
  id?: string | null;
  type: string;
}
```

Note: the referential stability (object reference) of all identifiers created by the
`store` is guaranteed. E.g. any data that results in the lookup of an identifier 
producing the same `lid` token will return the same `Identifier` instance. This is 
useful for being able to use identifiers for either `Map` or `WeakMap` cache solutions.

### Buckets

In an ideal world, the `lid` of each `Identifier` would be a `v4` [uuid](https://en.wikipedia.org/wiki/Universally_unique_identifier),
 making it practically unique in all contexts. However, due to requirements around design
 flexibility and performance we are only requiring that `Identifiers` be unique _within 
 their bucket_ for the data they are intended to reference.

Each underlying primitive will have its own bucket, as new primitives are formalized, new
 buckets will emerge. Initially we expect only a `record` bucket which aligns with today's
`IdentityMap` cache for `Record`s. Examples of future buckets may include a cache for 
`queries`, `documents`, `transactions`, `operations`, `errors`, `meta` or any number of other
 concepts that represent state required to be serializable.

In our ideal world, the `lid` would then be a `uuid-v4` that is practically unique across
 buckets and not just within. While we  will ship a minimal `uuid-v4` generator to be used
 for generating identifiers when needed on the client, generating large quantities of `uuid`s
 is cost prohibitive.  An early performance analysis suggests that a few thousand identifiers
 would bring a cost in the tens of milliseconds on powerful machines. When generating
 identifiers for data returned by an API, this cost would impede optimizations around
 rendering.

This cost is primarily due to the need to generate large quantities of random bytes: a cost
 that is necessarily cpu intensive. Additionally, many forms of data (such as `json-api` 
 resources) come with unique or nearly unique identifying information already (`type` + `id`,
`href` etc.).  Some APIs already make use of `v4` `uuid`s as IDs, and for these APIs it would
 make the most sense to implement a custom generation method to reuise these `id`s as `lid`s
 when present.
  
To balance performance with the requirements of `identity`, we are choosing what we feel is a
 *sensible default*.  Users for whom this default does not meet their requirements may override
the appropriate hooks to generate identifiers that do.

### Customizing Identifiers

For users wishing to provide increased guarantees around uniqueness and serializability
 of identifiers we provide the ability to configure how we generate and manage identifiers.

Given the guarantees around uniqueness and serializability that are required, this configuration
 applies to all `IdentifierCache` instances and meaning that it is shared across all buckets,
 store instances. Ideally the configuration does not change between application instances but
 for encapsulation purposes for fastboot and tests we allow and encourage repeated setup/teardown.
 Specifically, this recommendation is that in order to provide a strong guarantee of uniqueness
 and serializability, _identifiers generated by separate store or application instances but which
 represent the same data should result in the generation of the same `lid`_.

Indeed in this vein, we have not provided a mechanism for distinguishing what instance of a `store`
 has asked for an `Identifier` when multiple stores are present, but the `initializer` pattern
 recommended below does offer the ability to distinguish _per-application_. We do not recommend
 using this availability to affect your generation method.

Supplying custom `lid` generation can be done using `setIdentifierGenerationMethod`. Currently
 there is only one bucket (`record`) as discussed above, but we reserve the ability to add 
 additional buckets in the future.

Users should do any identifier customization within an instance-initialize prior to making use
 of the store. Given the more universal nature of this customization, we recommend ensuring that
 you consider the mechanics of multiple applications in fastboot or test application instance
 scenarios when instantiating and populating any secondary lookup tables or caches for identifiers.
 Weakmapping these lookup tables and caches to the application instance will accomplish this.
 An example is provided below.

```typescript
/*
  A method which can expect to receive various data as its first argument
  and the name of a bucket as its second argument. Currently the second
  argument will always be `record` data should conform to a `json-api`
  `Resource` interface, but will be the normalized json data for a single
  resource that has been given to the store.

  The method must return a unique (to at-least the given bucket) string identifier
  for the given data as a string to be used as the `lid` of an `Identifier` token.

  This method will only be called by either `getOrCreateIdentifier` or 
  `createIdentifierForNewRecord` when an identifier for the supplied data
  is not already known via `lid` or `type + id` combo and one needs to be
  generated or retrieved from a proprietary cache.

  `data` will be the same data argument provided to `getOrCreateIdentifier`
  and in the `createIdentifierForNewRecord` case will be an object with
  only `type` as a key.
*/
type GenerationMethod = (data: Object, bucket: string) => string;

/*
 A method which can expect to receive an existing `Identifier` alongside
 some new data to consider as a second argument. This is an opportunity
 for secondary lookup tables and caches associated with the identifier
 to be amended.

 This method is called everytime `updateRecordIdentifier` is called and
  with the same arguments. It provides the opportunity to update secondary
  lookup tables for existing identifiers.
  
 It will always be called after an identifier created with `createIdentifierForNewRecord`
  has been committed, or after an update to the `record` a `RecordIdentifier`
  is assigned to has been committed. Committed here meaning that the server
  has acknowledged the update (for instance after a call to `.save()`)

 If `id` has not previously existed, it will be assigned to the `Identifier`
  prior to this `UpdateMethod` being called; however, calls to the parent method
  `updateRecordIdentifier` that attempt to change the `id` or calling update
  without providing an `id` when one is missing will throw an error.
*/
type UpdateMethod = (identifier: StableIdentifier, newData: Object, bucket: string) => void;

/*
A method which can expect to receive an existing `Identifier` that should be eliminated
 from any secondary lookup tables or caches that the user has populated for it.
*/
type ForgetMethod = (identifier: StableIdentifier) => void;

/*
 A method which can expect to be called when the parent application is destroyed.

 If you have properly used a WeakMap to encapsulate the state of your customization
 to the application instance, you may not need to implement the `resetMethod`.
*/
type ResetMethod = () => void;

export function setIdentifierGenerationMethod(method: GenerationMethod): void {}

export function setIdentifierUpdateMethod(method: UpdateMethod): void {}

export function setIdentifierForgetMethod(method: ForgetMethod): void {}

export function setIdentifierResetMethod(method: ResetMethod): void {}
```

A simple custom generation method might be an increasing counter like below:

```typescript
import { setIdentifierGenerationMethod } form '@ember-data/store';

export function initialize(applicationInstance) {
  // note how `count` here is now scoped to the application instance
  // for our generation method by being inside the closure provided
  // by the initialize function
  let count = 0;

  setIdentifierGenerationMethod((resource: Resource) => {
    return resource.lid || `my-key-${count++}`;
  });
}

export default {
  name: 'configure-ember-data-identifiers',
  initialize
};
```

### Identifiers for Records

When discussing identifiers for records it is useful to be familiar with `json-api` 
interfaces for [ResourceObjects](https://jsonapi.org/format/#document-resource-objects) 
and [ResourceIdentifierObjects](https://jsonapi.org/format/#document-resource-identifier-objects).

Below, we expose a rough approximation of these interfaces as `Resource` including the
 potential presence of `lid`.

```typescript
import { Value as JSONValue } from 'json-typescript';

type JSONDict = { [k: string]: JSONValue };

export interface Resource {
  id: string;
  type: string;
  lid?: string;
  attributes?: JSONDict;
  relationships?: JSONDict;
  meta?: JSONDict;
}
```

We can access and generate identifiers for records using the following APIs available
 via the `identifierCache` on the `Store` and `StoreWrapper` classes.

```typescript
import Service from '@ember/service';

export interface Store {
  identifierCache: IdentifierCache
}

export interface StoreWrapper {
  identifierCache: IdentifierCache
}

export default class IdentifierCache extends Service {
  /*
   Returns the Identifier for the given Resource, creates one if it does not yet exist.

   Specifically this means that we:

   - validate the `id` `type` and `lid` combo against known identifiers
   - return an object with an `lid` that is stable (repeated calls with the same
    `id` + `type` or `lid` will return the same `lid` value)
   - this referential stability of the object itself is guaranteed
  */
  getOrCreateRecordIdentifier(resource: Resource): RecordIdentifier {}

  /*
   Returns a new Identifier for the supplied data. Call this method to generate
   an identifier when  a new resource is being created local to the client and
   potentially does not have an `id`.
  */
  createIdentifierForNewRecord({ type: string, id: string | null }): RecordIdentifier {}

  /*
   Provides the opportunity to update secondary lookup tables for existing identifiers
   
   Called with the attributes provided to createRecord after an identifier created with
   `createIdentifierForNewRecord` has been instantiated.

   Called again after an identifier created with `createIdentifierForNewRecord` has been
   committed, or a resource has received an update from the API.

   Assigns `id` to an `Identifier` if `id` has not previously existed; however,
   attempting to change the `id` or calling update without providing an `id` when
   one is missing will throw an error.
  */
  updateRecordIdentifier(identifier: RecordIdentifier, data: Resource): void;

  /*
   Provides the opportunity to eliminate an identifier from secondary lookup tables
   as well as eliminates it from ember-data's own lookup tables and book keeping.

   Useful when a record has been deleted and the deletion has been persisted and
   we do not care about the record anymore. Especially useful when an `id` of a
   deleted record might be reused later for a new record.
  */
  forgetRecordIdentifier(identifier: RecordIdentifier): void
}

// -- example uses

// ... for existing resources
let identifierA = identifierCache.getOrCreateRecordIdentifier({
  type: 'foo',
  id: '1'
}); // => { lid: 'some-unique-key-1324' }
let identifierB = identifierCache.getOrCreateRecordIdentifier({
  type: 'foo',
  id: '2',
  lid: '123a' 
}); // => { lid: '123a' }
let identifierC = identifierCache.getOrCreateRecordIdentifier({ 
  type: 'foo', 
  lid: '123b' 
}); // => { lid: '123b' }
let identifierD = identifierCache.getOrCreateRecordIdentifier({ 
  lid: '123c' 
}); // => { lid: '123c' }

// ... generating identifiers for newly created resources
// (this is something that likely only store.createRecord() should do)
let identifier1 = identifierCache.createIdentifierForNewRecord('foo'); // => { lid: 'some-random-unique-key-123a' }
let identifier2 = identifierCache.createIdentifierForNewRecord('foo'); // => { lid: 'some-random-unique-key-123b' }
let identifier3 = identifierCache.createIdentifierForNewRecord('bar'); // => { lid: 'some-random-unique-key-123c' }
```

### Updating new record Identifiers with more complete information

Called when an identifier has been generated for resource data prior to `id` being
 available for that resource and complete resource data is now available. `ember-data` 
 will automatically call this with the resolved payload after save for any newly created
 records. An `identifier` can only be updated once, and only when transitioning the 
 associated resource from a never-before-persisted to persisted state.

Udating provides the opportunity to update the primary and secondary lookup tables for the
 identifier. In the case of a RecordIdentifier that was created locally, it provides the
 ability to do a "one time only" upgrade of the identifier to assign an id.

```ts
IdentifierCache {
  updateRecordIdentifier(identifier: RecordIdentifier, data: Resource): void;
}
```

### Refreshing an `Identifier` When recycling an `id`

Occasionally some APIs re-use the same `id` for different `data`. Common scenarios for
 this include reusing the `id` of a previously deleted record for a new record, and less
 commonly a stable `id` to reference the "currently logged in user".

When this occurs, the existing `lid` needs the chance to be forgotten and a new `lid`
 generated. This method eliminates the identifier from our internal cache only. Caches
 associated with custom identifiier generation methods must be cleared by the implementors
 of those custom methods. Any data associated with the original `lid` should be purged from
 caches prior to calling this method. We leave it to follow up RFCs to provide
 infrastructure for safely and correctly eliminating records from caches.

```ts
IdentifierCache {
  forgetRecordIdentifier(identifier: RecordIdentifier): void
}
```

### Access from `record` instances

```typescript
export function recordIdentifierFor(record: object): RecordIdentifier {}
```

```ts
import { recordIdentifierFor } from '@ember-data/store';

// ...

// when you have a record instance
let identifier = recordIdentifierFor(record);

// from inside a record class after instantiation
class MyRecord {
  getIdentifier() {
    return recordIdentifierFor(this);
  }
}
```

Whether and how to access an identifier during instantiation of a record will be left
 for discussion as part of a different RFC for custom record classes.

### Polymorphism & "The Username Problem"

A common edge case that `Identifiers` enables end users to solve is when multiple pieces
 of identifying information should reference the same data.
 
For instance, when using single-table polymorphism (in which `ferrari` and `bmw` extend
 `car` and share a common `id` space) then `ferrari:1` and `bmw:2` are the same vehicles
  as `car:1` and `car:2`.

A similar problem presents for the scenario in which we know that we wish to reference a
`user` with a given `username`, but do not yet have access to the `id` for that user. In
 this case, `user:@jackson5` and `user:abc123` are the same user.

Today in these situations many users will encounter bugs resulting from there being two 
 records present in the cache instead of one. This problem can be solved with a custom 
 identifier generation method that is aware of an application's polymorphic associations
 or additional indexing requirements.

 For example, to solve the username problem we might do the following:

 ```typescript
import { setIdentifierGenerationMethod } from '@ember-data/store';

export const SECONDARY_IDENTIFIER_CACHE = new WeakMap();

export function initialize(applicationInstance) {
  let count = 0;
  const typeid_cache = {};
  const username_cache = {};

  /*
    Note: if you needed to share access to these caches elsewhere
    in the same applicationInstance, we could use a WeakMap to add
    them. Shown here for a more complete example.
  */
  SECONDARY_IDENTIFIER_CACHE.set(applicationInstance, {
    typeid: typeid_cache,
    username: username_cache
  });

  setIdentifierGenerationMethod((resource: Resource) => {
    let { type, id, lid } = resource;
    let username = (resource.type === 'user' 
      && resource.attributes
      && resource.attributes.username);
    let cacheKey, altCacheKey;

    if (lid) {
      // probably ensure username and id cache are populated first IRL
      return lid;
    }

    // handle the case where we do know the ID and have set the `lid` previously
    if (id) {
      cacheKey = `${type}:${id}`;

      if (lid = typeid_cache[cacheKey]) {
        return lid;
      }
    }

    // handle the cases where we have a username but we didn't know the ID yet
    if (username) {
      lid = username_cache[username];

      if (!lid) {
        lid =  `my-key-${count++}`;
        username_cache[username] = lid;
      }

      if (id) {
        typeid_cache[cacheKey] = lid;
      }
      return lid;
    }

    // handle everything else
    lid =  `my-key-${count++}`;
    typeid_cache[cacheKey] = lid;

    return lid;
  });
}

export default {
  name: 'configure-ember-data-identifiers',
  initialize
};
 ```

#### Handling Updates to Alternative Cache Keys
 
 Note that in our above example we treat `username` a stable, immutable alternative 
  primary-key. Some APIs allow users to change the value of such "unique keys" (`email`
  `phone` `username` being common examples).

 If your application enables such behavior, and these updates are not handled by the
  call to `updateRecordIdentifier` that occurs after a record is saved, in addition
  to manually calling `identifierCache.updateRecordIdentifier` with the desired patch
  you could also provide your own explicit method for doing so. An explicit method is
  great for ensuring the correct context for the granularity of this change. The timing
  of this update would be up to you (whether pre- or post- the mutation having been
  persisted to the server)

 Extending the example above:

 ```ts
 import { SECONDARY_IDENTIFIER_CACHE } from './initializers/configure-identifiers';

 export function updateUsernameForIdentifier(
   // application instance is the result of `getOwner`
   // on something like the `store` or a `component`
   owner: Owner, 
   identifier: Identifier,
   oldUsername: string,
   newUsername: string
   ) {
    const caches = SECONDARY_IDENTIFIER_CACHE.get(applicationInstance);
    const typeid_cache = caches.typeid;
    const username_cache = caches.username;

    if (username_cache[oldUsername] !== identifier.lid) {
      throw new Error('invalid update');
    }

    // you might want to continue mapping both old an new username
    //  here we decided not to.
    delete username_cache[oldUsername];

    username_cache[newUsername] = identifier.lid;
  }
 ```

 ### Identifier Stability

Identifiers handed to public APIs by `ember-data` will **always** be _referentially stable_
 Public `ember-data` APIs that **expect** an `Identifier` will normalize the object they 
 are given into the stable `Identifier` if it is not one already. This is done to allow
 for serialized identifiers and identifying information from the API to more easily be worked
 with without extra normalization effort.

Specifically, this means that as regards this RFC, `identifiers` that are structurally the same
 (meaning an object with the same `lid`) are treated as any other `identifier` regardless of
 whether the object is an identical reference to the object generated by the store previously.

Additionally, only the objects generated by the store will have additional debug information
 attached to them, as shown in the interfaces below.

```typescript
const IS_IDENTIFIER = Symbol('is-identifier');

// provided for additional debuggability
const DEBUG_CLIENT_ORIGINATED = Symbol('record-originated-on-client');
const DEBUG_IDENTIFIER_BUCKET = Symbol('identifier-bucket');

export interface StableIdentifier {
  lid: string;
  [IS_IDENTIFIER]: true;
  [DEBUG_IDENTIFIER_BUCKET]: string;
}

export interface StableRecordIdentifier extends StableIdentifier {
  id: string | null;
  type: string;
  [DEBUG_CLIENT_ORIGINATED]: boolean;
}
```

## How we teach this

Largely this is an internal feature, although one that power users will sometimes need
 access to. Additional guides should be created showing how identifiers may be used to
 solve common edge cases unique to given users. We have attempted to go into depth here
 with examples and documentation about the capabilities provided to make creating these
 additional resources as easy as possible.

Existing APIs that accept or provide `id` and `type` information will continue to do so
 unchanged.

## Drawbacks

- None, the performance characteristics of the setup here described may seem worse on
  allocation (object vs string identifier) but in reality they significantly improve our
  ability to reduce allocations, duplicate logic, and branching code paths throughout the
  library.

## Alternatives

A lengthy amount of discussion was had revolving around whether `Identifiers` shouldn't be
a simple `string` (e.g. just the `lid` portion).

The arguments for doing so revolve around `string` being referentially stable automatically,
and that we do not require an `object reference` for being able to use identifiers as keys in
`WeakMaps` both because `lid` is stable and because `ember-data` has need of controlling most
of the object lifecycle.

However, continuing to use an object wrapper, especially a stable object wrapper, comes with
a multitude of benefits, including:

* **debugging:** enhanced debugging by associating additional information with the
  identifier in development builds
* **debugging:** enhanced debugging by users being able to see type and id at a 
  glance when inspecting state.
* **bug prevention:** enforcement of the use of the identifier generation process 
  (to ensure lookup tables are properly populated)
* **debugging:** ability to tell at a glance that an identifier was properly processed
* **ergonomics:** closer alignment to `jsonapi` that makes it true that all `RecordIdentifiers`
  are also `ResourceIdentifierObjects`
* **ergonomics:** We can provide better `typescript` support for an object interface than a `string`
  given that any `string` would fit the `lid` interface but only correctly shaped objects (which includes
  the `Symbol` for `identifier`) would fit the correct object interface.
