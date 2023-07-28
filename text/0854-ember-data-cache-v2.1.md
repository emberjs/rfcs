---
stage: recommended
start-date: 2022-08-27T00:00:00.000Z
release-date: 2023-04-08T00:00:00.000Z
release-versions:
  ember-data: v4.12.0
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
teams:
  - data
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/854'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/923'
  recommended: 'https://github.com/emberjs/rfcs/pull/926'
---

<!--- 
Directions for above: 

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# EmberData Cache V2.1

## Summary

A rename of the Cache (RecordData) API with new restrictions and additional amendments
to the accepted #461 RecordData V2 spec. These alterations will further simplify EmberData
while increasing its flexibility and extensibility. In turn, this increased flexibility
provides the backbone for further design and exploration of new capabilities such as
QueryCache, GraphQL and an enhanced SSR and Rehydration Mode.

## Motivation

We originally designed RecordData as a public formalization of one of the core
responsibilities of `InternalModel`. This allowed applications that had needed to reach
into private API to achieve criticial feature support a public avenue for doing so. However,
even then we recognized that this was merely a bridge for supporting users while we
underwent heavier refactoring to rationalize the internals and better explore what was
needed in a long-term cache solution.

Between the efforts in ember-m3, orbit.js, our own implementation of RecordData V1 and V2,
and through the process of eliminating InternalModel we were able to come to a better
understanding of what capabilities the cache lacks today. Some of these learnings resulted
in [RFC #461](https://rfcs.emberjs.com/id/0461-ember-data-singleton-record-data). Others
were floated as internal proposals for some time, but needed more cleanup to happen
internally for us to be confident in their direction.

These changes look to either solve (or tee up) multiple key issues for our users. The below
discussion is not comprehensive nor exhaustive, but representative of the kinds of problems
we're looking to solve by effecting these changes.

For example: currently, if users want to avoid refetching a query they must design and place
their own cache in front of the store's query method. With the proposed changes, the cache
becomes capable of managing this responsibility. This turns out to be ideal for handling
paginated collections and GraphQL responses as well. Near Future RFCs will directly utilize
the changes in this RFC to bring these capabilities to users of EmberData.

Similarly, the query context and response documents contain information critical for the
correct handling of individual resources it contains. For instance, when fetching a related
collection via a link it is not a requirement of the `JSON:API` spec that the records in the
response have a relationship back to the requesting record of their own. Access to the original
response document is necessary to know both the membership and the ordering of the returned
relationship data. Today, EmberData iterates payloads and fills in the gaps of this missing
information by creating artificial records, but that ties us unnecessarily to the JSON:API spec
and results in unnecessarily high overhead for processing a response. A Cache which received
the full request context and document could properly update this relationship without any
additional overhead.

**Its the URL. Its always the URL**

Until now, EmberData has focused on being a Resource cache. While many of the changes here do
affect the Resource Caching APIs, **the introduction of Document caching is perhaps the most
substantive change in EmberData's long history.**

For many years discussion of a Document Cache focused on finding a solution to caching the
response of `Store.query` such that repeat calls did not require hitting network if that was
unnecessary. Typically these discussions got hung up on cache lifetimes and what to use as
cache-keys.

Then in late 2017 we had the insight that relationships also suffered from this lacking
capability, and began work on what was then [the collections RFC](https://github.com/runspired/rfcs/blob/ember-data-collections/text/0000-ember-data-collections.md).

Unfortunately, that RFC was premature, not for its proposed design but because it turned out to
require other Cache improvements it did not specify, was too coupled to JSON:API at a time when
we had already realized we wanted to support GraphQL, and required [Identifiers](https://github.com/emberjs/rfcs/pull/403)
to be fully realized.

As we continued to work through GraphQL support built over our R&D efforts in [ember-m3](https://github.com/hjdivad/ember-m3),
it became increasingly clear that if we were to truly make the data format opaque to the Store,
then the Cache must also be responsible for determining what in a response Document constituted
a Resource.

And so, with time, we embraced the lesson that Web Engineers have had to learn to embrace time and
time again: `use the URL`.

Caching by URL gives EmberData immediate access to a host of potential capabilities.

- `URL` is a unique identifier, including encoding information about pagination, sorting,
    filtering, and order
- `headers` associated with a URL typically contain cache directives a Cache implementation
    could utilize for data lifetimes
- `body` of a response to a URL can be in any format, not just JSON, and so caching blobs for
    images, xml documents like SVG, video streams, etc. becomes feasible (these are capabilities
    `Identifiers` previously allowed but only at a `Resource` level which made it not all that
    useful)

Caching by URL gives EmberData enhanced abilities when the response Document and Cache 
implementation are aligned in format. For instance, with JSON:API the top level of a response 
will often contain

- `links` (URLs!) for additional pages related to the current query (`next`, `prev` and so on)
- `meta` that may contain information around total count still to be loaded
- `order` implicitly the order of the `data` array in a collection response *is* valuable 
    information for sorting
- `included` information about what was and was not loaded for a particular request, which might
    be used to construct a more robust secondary fetch. For instance, instead of loading `user`
    and then later loading individual resources or types of resources associated to that user, 
    you might be able to use this information to construct a request for exactly the set of 
    missing data to fill out the graph once it becomes needed.

A Document centric cache is also essential to the GraphQL story where while the Document as 
a whole is a composition of Resources the entry point is the query itself.

Our vision with EmberData is one of composable primitives flexible enough to handle organizing
and coordinating desired capabilities. It is clear to us now that to embrace that vision requires
embracing the URL.

But this new Cache brings so much more! In a move atypical for EmberData, we
are proposing new public APIs here that are not simply a codification of private
APIs we've had years to explore. Our reasoning is straightforward: to encourage
exploration within constraints.

**Streaming the Future**

The addition of Documents to the cache provides the ability to cache all information
necessary for reconstruction of what was delivered to the UI. This affords a number
of interesting opportunities around cache dump and cache restore, including among other
things for offline support, rehydration, and background workers.

We don't want to stand in the way of such exploration while waiting to dream of the
perfect API, but we would like to provide a framework for it that helps cache's know
what to implement to be maximally compatible with solutions the community may dream up.

To this end, we are introducing with minimal constraints and guidance streaming APIs
for dumping, restoring and patching the cache. The format is almost entirely opaque,
because typically the optimal serialization form will be as-close as possible to the
form the cache stores this information in regularly. Where it is not opaque is merely
the constraint that the format needs to be chunkable. Of course a Cache could choose
to treat the entire state as one chunk, but hopefully by encouraging chunking from the
beginning Cache implementations will position themselves to take maximal advantage of
the very good native streaming and buffer APIs.

**Forking the Past**

The final change to this version of the Cache is that it will support forking. Cache
forking is a technique we are ~~stealing~~ (uh) ~~borrowing~~ (err) *learning* from [Orbit.js](https://github.com/orbitjs/orbit/blob/2df79c5d67f2311c3bdc84941a18cc6896668f25/website/docs/memory-sources.md#forking-memory-sources)

On the surface, forking has been described to us by some folks as a solution without a
problem. We disagree. Forking is the right sort of boring solution that has the power
to be disruptive in just how simple and boring it is. Why is that? Instead of solving
one problem, it is a general solve to a large set of problems.

Need to load a lot of modeled relational data for a single route or component, but for
memory concerns want to unload it when you exit that route or dismount that component?
Use a fork! The fork will allow you load additional data related to data you have already
loaded previously, then safely and quickly toss it away later when you no longer need it.

Need to edit a lot of relational data and save it in one go? Use a fork! If you later
want to discard the edits, instead of trying to keep track of what changed, just toss
away the fork. Or, if you want to persist it, save all changes to the fork in one single
go.

Want to have some data loaded for all routes, but have other routes or engines manage their
own data but still correctly be able to reference the global data? That's right, use forks!

Want to be able to edit a whole lot of data and serialize the changes for local storage for
a save/restore feature for a form, but don't want to pollute those records for where they
appear elsewhere in your UI? Use a fork!

Is it a bird, is it a plane? It might sound like a knife and feed you as easily as a spoon...
but its a fork!

### The Grand Vision (Abbreviated)

This RFC codifies the direction and vision that the EmberData team has been building
towards in our long-term strategy since early in my tenure on the team. To anyone who
has been in these discussions the ideas presented here are not new, and we already know
how they fit neatly into that vision; but for the community at large many of these
changes may still be a surprise.

This RFC is not "the grand vision", but it is an integral piece. Without it, changes we
have planned for the presentation, network and store primitives will not be possible.

Above, I have laid out several key use-cases and motivations for the changes herein,
but the observant will recognize that this RFC does not expose APIs or specify how
the store, network, or current model primitives might make use of these cache APIs. We
leave each of these to their own RFCs – coming very soon – but for the sake of adequate
discussion herein we outline the direction we expect those RFCs to take.

**The APIs shown in this section are not a proposed design but representative of the overall vision for EmberData in the near future** 

**`From Fetch to Fully Distributed Replica Database`**

For the network layer, we see the introduction of `FetchManager`, a managed flow
for making requests that allows registration of middleware to handle fulfillment
or decoration of the request or response. This service would thereby make headers
and auth management easy to handle, letting apps that previously had multiple
unorganized services or ad-hoc fetch handlers to unify over one conventional flow.

This managed `Fetch` flow would (by default) not be integrated with the Store, thereby
skipping the cache and providing access to raw responses.

```ts
class FetchManager {
  registerMiddlewares(wares: Middleware[]);

  async request(req: RequestOptions): RequestFuture;
}
```

For the store, we see the addition of a new API – `Store.request` – overwhich all
existing store methods will be rebuilt as "convenience macros". This delegates to
the above `FetchManager`, but passes the Store's cache (which is potentially a fork)
as a reserved arg, and which manages passing the result to the Cache and instantiating
any records or other presentation classes required in the return value.

```ts
class Store {
  async request(req): RequestFuture;
}
```

```ts
import { gql } from '@ember-data/graphql';

const USER_QUERY = gql`query singleUser($id: Id) {
  user(id: $id) {
    name
    friends {
      name
    }
  }
}`;

class extends Route {
  @service store;

  async model({ id }) {
    const user = await this.store.request(USER_QUERY({ id }, { cache: { backgroundReload: true } }));

    return user.data;
  }
}
```

Similarly, we see request-builders (like the graphql example above) becoming the primary mechanism
by which folks query for data. This ensures URLs are available for the cache, illuminates the mental
model of `fetch => fully distributed replica Database`, simplifies the mental model of EmberData in
general by removing the mystery of "how" and "when" data is fetched, and encourages a pattern that
can benefit from static analysis and build-time optimizations (such as pre-built GraphQL queries).

```ts
import { buildUrlForFindRecord } from '@ember-data/request-builders';

class extends Route {
  @service store;

  async model({ id }) {
    const user = await this.store.request({
      url: buildUrlForFindRecord({
        type: 'user',
        id,
        incude: 'friends,company,parents,children,pets'
      })
    });

    return user.data;
  }
}
```

The Store will also gain APIs for forking and for peeking documents.

For modeling data, we see a story that is immutable-by-default. All edits run through forks,
which occurs by calling `Store.fork` which would similarly setup its cache via `Cache.fork`.

A major benefit of this approach is that it gives application developers the ability to choose
the specifics of the optimistic or pessimistic timing of when the "remote" state is updated.

An app might eagerly update the primary store on every edit, or it might update it optimistically
once a save has initiated, or it may wait only until the server has confirmed the update to do so.

In this way, global state always reflects the developer's wishes for the specific application:
whether that is the persisted state or the mutated state.

Of course, because these APIs are opaque and the Store is merely playing the role of coordinator,
libraries can choose to create all sorts of pairing of capabilities and patterns between cache
and presentation, and we will be able to maintain the existing `@ember-data/model` story for some
time as a "legacy" approach that users can install if they are not yet ready to upgrade to these
newer features and patterns.

**Reimagining the Install and Learning Experience**

The sum total of these changes (this RFC and planned RFCs) leads to a new default story for
EmberData. For this reason I have at times in jest referred to it as "The Grand EmberData
Modules Disunification RFC".

For some time now the inclusion of EmberData's `ember-data` package in the `ember-cli` blueprint
has made increasingly less sense: not because we don't want EmberData to be the default, but
because we don't see there existing just one "EmberData way".

An ideal install experience would likely focus on capabilities and requirements, a guided install
where running `npx ember-data` walks you through choices like `JSON:API` vs `GraphQL`, "just fetch"
or all the way to "distributed replica" along the way showing what packages you will need and in
the end installing them and wiring them together to get you started.

An ideal learning experience would likely take a similar path; documentation that guides the user
incrementally from fetch to full-story, and a tutorial app that shows using EmberData for just
fetch+ and then as needs in the tutorial app expand adding in additional packages and capabilities
to match.

And so armed with this context, let's dive in.

## Detailed design

This RFC proposes five substantive changes.

### 1. `RecordData` becomes `Cache`

First, the cache interface is renamed from `RecordData` to `Cache`. This is to reflect its upgraded responsibilities from handling only Resource data to also handling the caching of Documents. Additionally, `Cache` implementations MUST be Singletons.

The following APIs are affected by this change.

 - `StoreWrapper.recordDataFor` is deprecated without replacement. Since the cache is a Singleton it will already contain a reference to itself.

 - On `Store`
   - `Store.instantiateRecord` will receive only two args: `identifier` and `createArgs`
    - the removed args are `recordDataFor` and `notificationManager` which are now available as the store's `cache` and `notifications` properties
   - The notification manager (already public via instantiateRecord) is made accessible via `Store.notifications`
   - `Store.createRecordDataFor` is deprecated in favor of `Store.createCache`
   - `Store.createCache` and `Store.cache` are added with the following signatures
 
   ```ts
   class Store {
    /**
     * This hook allows for supplying a custom cache
     * implementation following the Cache spec.
     * This hook will only be called once by the Store
     * the first time cache access is needed.
     *
     * For interopability, there are multiple avenues
     * for composing caches together to support a range
     * of capabilities.
     *
     * When extending a store with an alternative cache
     * you may super this meethod. Alternatively you may
     * extend another public Cache implementation, or
     * manually instantiate and wrap another cache implentation as a delegate.
     *
     * You should not call this method yourself.
     * 
     * @method createCache (hook)
     * @public
     * @returns {Cache} a new Cache instance
     */
      createCache(store: StoreWrapper): Cache;

      /**
       * The store's Cache instance. Instantiates it
       * if needed.
       *
       * @property {Cache} cache
       * @public
       */
      cache: Cache;

      /**
       * Provides access to the NotificationManager instance
       * for this store.
       *
       * The NotificationManager can be used to subscribe to changes
       * for any identifier.
       *
       * @property {NotificationManager} notifications
       * @public
       */
      notifications: NotificationManager;


      /**
       * A hook which an app or addon may implement. Called when
       * the Store is attempting to create a Record Instance for
       * a resource.
       *
       * This hook can be used to select or instantiate any desired
       * mechanism of presentating cache data to the ui for access
       * mutation, and interaction.
       *
       * @method instantiateRecord (hook)
       * @param identifier
       * @param createRecordArgs
       * @param recordDataFor
       * @param notifications
       * @returns A record instance
       * @public
       */
      instantiateRecord(
        identifier: StableRecordIdentifier,
        createRecordArgs: { [key: string]: unknown },
      ): RecordInstance;
   }
   ```

 - From a Package Perspective
   - `@ember-data/record-data` will be rebranded as `@ember-data/json-api` and the Cache
      implementation will be publically available as `import Cache from '@ember-data/json-api';`
      This means consumers are free to extend this implementation if desired, though this is not recommended.
   - A new package, `@ember-data/graph`, will be introduced, it will currently contain no public
      API exports. This is done so that EmberData may experiment with providing additional
      official cache implementations that also make use of the primitives we are designing for 
      managing relationships between resources. This abstract utility will become public after
      some additional iteration.

### 2. Simplification of Resource Cache API

These changes are *instead of* the changes in the original RecordData V2 RFC.

> **Note** for a transitionary period `Store.createRecordDataFor` will still be invoked if present and
> it should return the singleton cache instance if the RecordData has been upgraded to Cache 2.1.
> This should ease the transition from V1 to V2 and provide interop between.

Below is the finalized "v2.1" API for Resources. All existing methods and signatures not
contained herein are deprecated.

<table>
<thead>
  <tr><th>Cache APIs</th><th>Associated Types</th></tr>
</thead>
<tbody>
<tr>
<td valign="top">

```ts
export interface Cache {
  /**
   * The Cache Version that 
   * this implementation implements. Defaults
   * to '1' if not defined.
   * 
   * @type {'1'|'2'}
   * @property version
   */
  version: '2';

  // Resource Cache APIs
  // ===================

  /**
   * Push resource data from a remote source into the cache for this identifier
   *
   * @method upsert
   * @public
   * @param identifier
   * @param data
   * @param hasRecord
   * @returns {void | string[]} if `hasRecord` is true then calculated key changes should be returned
   */
  upsert(
    identifier: StableRecordIdentifier,
    data: ResourceBlob,
    hasRecord?: boolean
  ): void | string[];

  /**
   * [LIFECYCLE] Signal to the cache that a new record has been instantiated on the client
   *
   * It returns properties from options that should be set on the record during the create
   * process. This return value behavior is deprecated.
   *
   * @method clientDidCreate
   * @public
   * @param identifier
   * @param createArgs
   */
  clientDidCreate(
    identifier: StableRecordIdentifier,
    createArgs?: Record<string, unknown>
  ): Record<string, unknown>;

  /**
   * [LIFECYCLE] Signals to the cache that a resource
   * will be part of a save transaction.
   *
   * @method willCommit
   * @public
   * @param identifier
   */
  willCommit(
    identifier: StableRecordIdentifier
  ): void;

  /**
   * [LIFECYCLE] Signals to the cache that a resource
   * was successfully updated as part of a save transaction.
   *
   * @method didCommit
   * @public
   * @param identifier
   * @param data
   */
  didCommit<T>(
    identifier: StableRecordIdentifier,
    data: StructuredDataDocument<T>
  ): void;

  /**
   * [LIFECYCLE] Signals to the cache that a resource
   * was update via a save transaction failed.
   *
   * @method commitWasRejected
   * @public
   * @param identifier
   * @param errors
   */
  commitWasRejected(
    identifier: StableRecordIdentifier,
    errors?: StructuredErrorDocument
  ): void;

  /**
   * [LIFECYCLE] Signals to the cache that all data for a resource
   * should be cleared.
   * 
   * This method is a candidate to become a mutation
   *
   * @method unloadRecord
   * @public
   * @param identifier
   */
  unloadRecord(
    identifier: StableRecordIdentifier
  ): void;

  // Flexible Resource APIs
  // ======================
  // These APIs have additional
  // signatures detailed in
  // other sections

  /**
   * Perform an operation on the cache to update the remote state.
   *
   * Note: currently the only valid operation is a MergeOperation
   * which occurs when a collision of identifiers is detected.
   *
   * @method patch
   * @public
   * @param op the operation to perform
   * @returns {void}
   */
  patch(
    op: Operation
  ): void;

  /**
   * Update resource data with a local mutation.
   * Currently supports operations on relationships
   * only.
   *
   * @method mutate
   * @public
   * @param operation
   */
  mutate(
    mutation: Mutation
  ): void;

  /**
   * Peek resource data from the Cache.
   * 
   * In development, if the return value
   * is JSON the return value
   * will be deep-cloned and deep-frozen
   * to prevent mutation thereby enforcing cache
   * Immutability.
   * 
   * This form of peek is useful for implementations
   * that want to feed raw-data from cache to the UI
   * or which want to interact with a blob of data
   * directly from the presentation cache.
   * 
   * An implementation might want to do this because
   * de-referencing records which read from their own
   * blob is generally safer because the record does
   * not require retainining connections to the Store
   * and Cache to present data on a per-field basis.
   * 
   * This generally takes the place of `getAttr` as
   * an API and may even take the place of `getRelationship`
   * depending on implementation specifics, though this
   * latter usage is less recommended due to the advantages
   * of the Graph handling necessary entanglements and
   * notifications for relational data.
   * 
   * @method peek
   * @public
   * @param identifier
   * @returns {ResourceBlob | null} the known resource data
   */
  peek(
    identifier: StableRecordIdentifier
  ): ResourceBlob | null;

  // Granular Resource Data APIs
  // ===========================

  /**
   * Retrieve the data for an attribute from the cache
   *
   * @method getAttr
   * @public
   * @param identifier
   * @param field
   * @returns {unknown}
   */
  getAttr(
    identifier: StableRecordIdentifier,
    field: string
  ): unknown;
  
  /**
   * Mutate the data for an attribute in the cache
   * 
   * This method is a candidate to become a mutation
   *
   * @method setAttr
   * @public
   * @param identifier
   * @param field
   * @param value
   */
  setAttr(
    identifier: StableRecordIdentifier,
    field: string,
    value: unknown
  ): void;

  /**
   * Query the cache for the changed attributes of a resource.
   *
   * @method changedAttrs
   * @public
   * @deprecated
   * @param identifier
   * @returns { <field>: [<old>, <new>] }
   */
  changedAttrs(
    identifier: StableRecordIdentifier
  ): Record<string, [unknown, unknown]>;
  
  /**
   * Query the cache for whether any mutated attributes exist
   *
   * @method hasChangedAttrs
   * @public
   * @param identifier
   * @returns {boolean}
   */
  hasChangedAttrs(
    identifier: StableRecordIdentifier
  ): boolean;

  /**
   * Tell the cache to discard any uncommitted mutations to attributes
   *
   * This method is a candidate to become a mutation
   * 
   * @method rollbackAttrs
   * @public
   * @param identifier
   * @returns {string[]} the names of fields that were restored
   */
  rollbackAttrs(
    identifier: StableRecordIdentifier
  ): string[];

  /**
   * Query the cache for the current state of a relationship property
   *
   * @method getRelationship
   * @public
   * @param identifier
   * @param field
   * @returns resource relationship object
   */
  getRelationship(
    identifier: StableRecordIdentifier,
    field: string
  ): Relationship;


  // Resource State
  // ===============

  /**
   * Update the cache state for the given resource to be marked
   * as locally deleted, or remove such a mark.
   *
   * This method is a candidate to become a mutation
   * 
   * @method setIsDeleted
   * @public
   * @param identifier
   * @param isDeleted {boolean}
   */
  setIsDeleted(
    identifier: StableRecordIdentifier,
    isDeleted: boolean
  ): void;

  /**
   * Query the cache for any validation errors applicable to the given resource.
   *
   * @method getErrors
   * @public
   * @param identifier
   * @returns {ValidationError[]}
   */
  getErrors(
    identifier: StableRecordIdentifier
  ): ValidationError[];

  /**
   * Query the cache for whether a given resource has any available data
   *
   * @method isEmpty
   * @public
   * @param identifier
   * @returns {boolean}
   */
  isEmpty(
    identifier: StableRecordIdentifier
  ): boolean;

  /**
   * Query the cache for whether a given resource was created locally and not
   * yet persisted.
   *
   * @method isNew
   * @public
   * @param identifier
   * @returns {boolean}
   */
  isNew(
    identifier: StableRecordIdentifier
  ): boolean;

  /**
   * Query the cache for whether a given resource is marked as deleted (but not
   * necessarily persisted yet).
   *
   * @method isDeleted
   * @public
   * @param identifier
   * @returns {boolean}
   */
  isDeleted(
    identifier: StableRecordIdentifier
  ): boolean;

  /**
   * Query the cache for whether a given resource has been deleted and that deletion
   * has also been persisted.
   *
   * @method isDeletionCommitted
   * @public
   * @param identifier
   * @returns {boolean}
   */
  isDeletionCommitted(
    identifier: StableRecordIdentifier
  ): boolean;

}
```

</td>
<td valign="top">

<details>
  <summary>Types</summary>



```ts
// The ResourceBlob is an opaque type that must
// satisfy two constraints.
// (1) it should be possible for the IdentifierCache
// to be able to generate a RecordIdentifier for it
// whether by default or due to configuration.
// (2) it should be in a format expected by the Cache.
// This format is Cache declared.
// 
// this Opaqueness allows arbitrary storage of any
// serializable / transferable state including such things
// as Buffers and Strings.
type ResourceBlob = unknown;

// a "Stable" RecordIdentifier means
// that the object reference is known to
// the IdentifierCache, and as such
// referential integrity may be used
// to key by reference if desired.
interface StableRecordIdentifier {
  type: string;
  lid: string;
  id: string | null;
}

// An error relating to a Resource
// Received when attempting to persist
// changes to that resource.
// 
// considered "opaque" to the Store itself.
//
// Currently we restrict Errors to being
// shaped in JSON:API format; however,
// this is a restriction we will willingly
// recede if desired. So long as the
// presentation layer and the cache and the
// network layer are in agreement about the
// schema of these Errors, then EmberData
// has no reason to enforce this shape.
interface ValidationError {
  title: string;
  detail: string;
  source: {
    pointer: string;
  };
}

interface Op {
  op: string;
}

// Occasionally the IdentifierCache
// discovers that two previously thought
// to be distinct Identifiers refer to
// the same ResourceBlob. This Operation
// will be performed giving the Cache the
// change to cleanup and merge internal
// state as desired when this discovery
// is made.
interface MergeOperation extends Op {
  op: 'mergeIdentifiers';
  // existing
  record: StableRecordIdentifier; 
  // new
  value: StableRecordIdentifier;
}

// An Operation is an action that updates
// the remote state of the Cache in some
// manner. Additional Operations will be
// added in the future.
type Operation = MergeOperation;

// A Mutation is an action that updates
// the local state of the Cache in some
// manner.
// Most Mutations are in theory also
// Operations; with the difference being
// that the change should be applied as
// "local" or "dirty" state instead of
// as "remote" or "clean" state.
// 
// Note: this RFC does not publicly surface
// any of the mutations listed here as
// "operations", though the (private) Graph
// already expects and utilizes these.
// and we look forward to an RFC that makes
// the Graph a fully public API.
type Mutation =
  | ReplaceRelatedRecordsMutation
  | ReplaceRelatedRecordMutation
  | RemoveFromRelatedRecordsMutation
  | AddToRelatedRecordsMutation
  | SortRelatedRecordsMutation;


// Note: in v1 data could be a ResourceIdentifier, now
// we request that it be in the stable form already.
interface ResourceRelationship {
  data?: StableRecordIdentifier | null;
  meta?: Dict<JSONValue>;
  links?: Links;
}

// Note: in v1 data could be a ResourceIdentifier, now
// we request that it be in the stable form already.
interface CollectionRelationship {
  data?: StableRecordIdentifier[];
  meta?: Dict<JSONValue>;
  links?: PaginationLinks;
}

type Relationship = ResourceRelationship | CollectionRelationship;

```

</details>

<details>
  <summary>Mutations</summary>



```ts

interface AddToRelatedRecordsMutation {
  op: 'addToRelatedRecords';
  record: StableRecordIdentifier;
  field: string;
  value: StableRecordIdentifier | StableRecordIdentifier[];
  index?: number;
}

interface RemoveFromRelatedRecordsMutation {
  op: 'removeFromRelatedRecords';
  record: StableRecordIdentifier;
  field: string;
  value: StableRecordIdentifier | StableRecordIdentifier[];
  index?: number;
}

interface ReplaceRelatedRecordMutation {
  op: 'replaceRelatedRecord';
  record: StableRecordIdentifier;
  field: string;
  // never null if field is a collection
  value: StableRecordIdentifier | null; 
  // if field is a collection,
  //  the value we are swapping with
  prior?: StableRecordIdentifier;
  index?: number;
}

interface ReplaceRelatedRecordsMutation {
  op: 'replaceRelatedRecords';
  record: StableRecordIdentifier;
  field: string;
  // the records to add. If no prior/index
  //  specified all existing should be removed
  value: StableRecordIdentifier[];
  // if this is a "splice" the
  //  records we expect to be removed
  prior?: StableRecordIdentifier[];
  // if this is a "splice" the
  //   index to start from
  index?: number;
}

interface SortRelatedRecordsMutation {
  op: 'sortRelatedRecords';
  record: StableRecordIdentifier;
  field: string;
  value: StableRecordIdentifier[];
}

```

</details>

</td>
</tr>
</tbody>
</table>

A key takeaway from these changes should be that generally the Resource API is evolving away
from hyper-granular methods for operating on the cache towards general-purpose methods
that allow for customized granularity and can be extended via additional sigantures (operations)
instead of by adding new methods.

This is partly-due to the cache evolving to handle more than just resources, but it is also due
to us desiring to eliminate non-opaque-schema from the core APIs. EmberData should not need to
care whether some field is an Attribute or a Relationship. It needs to manage the flow of mutations
and queries, but it need not define their specificity nor shoe-horn the mechanics into artificial
constraints.

We expect further evolution in this area in the future, mostly in the direction of removing
single-purpose methods towards operational flows.

### 3. Introduction of Document Cache API

The design for caching documents focuses on solving three key constraints.

1) It should be possible to serialize the cache, and so the data handed to cache should
    be in a serialized form.
2) It should be possible to rebuild/retrieve the raw response via peek so that if
    desired the Cache could be used as little more than (for instance) an in-memory
    JSON store.
3) The cache should have access to serializable information about the request and response
    that may be required for proper cache storage, management or desired for later access.

<table>
<thead>
<tr><th>Cache APIs</th><th>Associated Types</th></tr>
</thead>
<tbody>
<tr>
<td>

```ts
class Cache {
  /**
   * Cache the response to a request
   * 
   * Unlike `store.push` which has UPSERT
   * semantics, `put` has `replace` semantics similar to
   * the `http` method `PUT`
   * 
   * the individually cacheable resource data it may contain
   * should upsert, but the document data surrounding it should
   * fully replace any existing information
   * 
   * Note that in order to support inserting arbitrary data
   * to the cache that did not originate from a request `put`
   * should expect to sometimes encounter a document with only
   * a `data` member and therefor must not assume the existence
   * of `request` and `response` on the document.
   * 
   * @method put
   * @param {StructuredDocument} doc
   * @returns {ResourceDocument}
   * @public
   */
  put(doc: StructuredDocument): ResourceDocument;

  /**
   * Update the "remote" or "canonical" (persisted) state of the Cache
   * by merging new information into the existing state.
   *
   * @method patch
   * @param {Operation} op
   * @returns {void}
   * @public
   */
  patch(
    op: Operation
  ): void;

  /**
   * Update the "local" or "current" (unpersisted) state of the Cache
   * 
   * @method mutate
   * @param {Mutation} mutation
   * @returns {void}
   * @public
   */
  mutate(
    mutation: Mutation
  ): void;

  /**
   * Peek the Cache for the existing data associated with
   * a StructuredDocument
   * 
   * @method peek
   * @param {StableDocumentIdentifier}
   * @returns {ResourceDocument | null}
   * @public
   */
  peek(
    identifier: StableDocumentIdentifier
  ): ResourceDocument | null;

  /**
   * Peek the Cache for the existing request data associated with
   * a cacheable request
   * 
   * @method peekRequest
   * @param {StableDocumentIdentifier}
   * @returns {StableDocumentIdentifier | null}
   * @public
   */
  peekRequest(identifier: StableDocumentIdentifier): StructuredDocument<ResourceDocument> | null;

  didCommit<T>(
    identifier: StableRecordIdentifier,
    data: StructuredDataDocument<T>
  ): void;

  commitWasRejected(
    identifier: StableRecordIdentifier,
    errors?: StructuredErrorDocument
  ): void;
}
```


</td>
<td>

```ts

interface RequestInfo extends Request {
  disableTestWaiter?: boolean;
  /*
   * data that a handler should convert into
   * the query (GET) or body (POST)
   */
  data?: Record<string, unknown>;
  /*
   * options specifically intended for handlers
   * to utilize to process the request
   */
  options?: Record<string, unknown>;
}

interface ResponseInfo {
  readonly headers: ImmutableHeaders; // to do, maybe not this?
  readonly ok: boolean;
  readonly redirected: boolean;
  readonly status: number;
  readonly statusText: string;
  readonly type: string;
  readonly url: string;
}

interface StructuredDataDocument<T> {
  request: RequestInfo;
  response: Response | ResponseInfo | null;
  content: T;
}
interface StructuredErrorDocument extends Error {
  request: RequestInfo;
  response: Response | ResponseInfo | null;
  error: Error;
  content?: unknown;
}

type StructuredDocument<T> = StructuredDataDocument<T> | StructuredErrorDocument;

// must have one of meta, data or error
interface ResourceDocument {
  // the url or cache-key associated with the structured document
  lid: string; 
  links?: Links;
  meta?: Meta;
  data?: StableRecordIdentifier | StableRecordIdentifier[] | null;
  error?: Error;
}

```

</td>
</tr>
</tbody>
</table>

#### Notice about RequestStateService

Currently the `RequestStateService` which provides access to promise information about
save/fetch requests for individual resources will not be altered. However, the response
cache it provides will now differ in structure from the `StructuredDocument` provided to the
cache, and relying on it for more than querying the status of a request will prove brittle
as we expect this to undergo further design work to align with `StructuredDocument` in the
near future as we introduce a new design for managing requests.


#### Changes to `store.push`

Historically, `store.push` received a `JSON:API` document from which it decomposed resources
from within `data` and `included` which it then individually inserted into the cache. As well,
`store.push` was the primary mechanism by which data would load into the cache following any
request.

Beginning with this new cache implementation, this role will be significantly reduced. Instead,
responses from requests will be passed as `StructuredDocuments` directly to the cache for the
cache to decompose as it sees fit.

`store.push` will remain, however it too will see two significant changes.
  1) It will no longer decompose the data it receives, instead pushing that
      data (`T`) into the cache via `cache.put({ data: T })`
  2) It will no longer be required that the data given to `push` be in `JSON:API` format,
      the new requirement will be that it be in the format expected by the configured `Cache`.

### 4. Introduction of Streaming Cache API

Cache implementations should implement two methods to support streaming
SSR and AOT Hydration.

`cache.dump` should return a stream of the cache's contents that can be
provided to the same cache's `hydrate` method. The only requirement is
that a `Cache` should output a stream from `cache.dump` that it can also import
via `cache.hydrate`. The opaque nature of this contract allows cache implementations
the flexibility to experiment with the best formats for fastest restore.

`cache.dump` returns a promise resolving to this stream to allow the cache the
opportunity to handle any necessary async operations before beginning the stream.

`cache.hydrate` should accept a stream of content to add to the cache, and return
a `Promise` the resolves when reading the stream has completed or rejects if for
some reason `hydrate` has failed. Currently there is no defined behavior for
recovery when `hydrate` fails, and caches may handle a failure however they see fit.

A key consideration implementations of `cache.hydrate` must make is that `hydrate`
should expect that it may be called in two different modes: both during initial
hydration and to hydrate additional state into an already booted application.



```ts
class Cache {
  /**
   * Serialize the entire contents of the Cache into a Stream
   * which may be fed back into a new instance of the same Cache
   * via `cache.hydrate`.
   * 
   * @method dump
   * @returns {Promise<ReadableStream>}
   * @public
   */
  async dump(): Promise<ReadableStream<unknown>>;
 
  /**
   * hydrate a Cache from a Stream with content previously serialized
   * from another instance of the same Cache, resolving when hydration
   * is complete.
   * 
   * This method should expect to be called both in the context of restoring
   * the Cache during application rehydration after SSR **AND** at unknown
   * times during the lifetime of an already booted application when it is
   * desired to bulk-load additional information into the cache. This latter
   * behavior supports optimizing pre/fetching of data for route transitions
   * via data-only SSR modes.
   * 
   * @method hydrate
   * @param {ReadableStream} stream
   * @returns {Promise<void>}
   * @public
   */
  async hydrate(stream: ReadableStream<unknown>): void;
}
```


### 5. Introduction of Cache Forking

Cache implementations should implement three methods to support
 store forking. While the mechanics of how a Cache chooses to
 fork are left to it, forks should expect to live-up to the following
 constraints.

1. A parent should never retain a reference to a child.
2. A child should never mutate/update the state of a parent.

> Note: when saving state on a `Store`, the `store` is provided as the
> first argument to Adapter and Serializer methods, thereby allowing
> these class instances to access information about the current `Store`
> which may not be the same instance as the singleton `Store`
> injectible as a service. This is in keeping with the same existing design
> we have today for these classes to support multiple unique top-level stores.

```ts
class Store {
  /**
   * Create a fork of the Store starting
   * from the current state.
   * 
   * @method fork
   * @public
   * @returns Promise<Store>
   */
  async fork(): Promise<Store>;

  /**
   * Merge a fork back into a parent Store
   * 
   * @method merge
   * @param {Store} store
   * @public
   * @returns Promise<void>
   */
  async merge(store: Store): void;
}
```

```ts
class Cache {
  /**
   * Create a fork of the cache from the current state.
   * 
   * Applications should typically not call this method themselves,
   * preferring instead to fork at the Store level, which will
   * utilize this method to fork the cache.
   * 
   * @method fork
   * @public
   * @returns Promise<Cache>
   */
  async fork(): Promise<Cache>;

  /**
   * Merge a fork back into a parent Cache.
   * 
   * Applications should typically not call this method themselves,
   * preferring instead to merge at the Store level, which will
   * utilize this method to merge the caches.
   * 
   * @method merge
   * @param {Cache} cache
   * @public
   * @returns Promise<void>
   */
  async merge(cache: Cache): void;

  /**
   * Generate the list of changes applied to all
   * record in the store.
   * 
   * Each individual resource or document that has
   * been mutated should be described as an individual
   * `Change` entry in the returned array.
   * 
   * A `Change` is described by an object containing up to
   * three properties: (1) the `identifier` of the entity that
   * changed; (2) the `op` code of that change being one of
   * `upsert` or `remove`, and if the op is `upsert` a `patch`
   * containing the data to merge into the cache for the given
   * entity.
   * 
   * This `patch` is opaque to the Store but should be understood
   * by the Cache and may expect to be utilized by an Adapter
   * when generating data during a `save` operation.
   * 
   * It is generally recommended that the `patch` contain only
   * the updated state, ignoring fields that are unchanged
   * 
   * ```ts
   * interface Change {
   *  identifier: StableRecordIdentifier | StableDocumentIdentifier;
   *  op: 'upsert' | 'remove';
   *  patch?: unknown;
   * }
   * ```
   * 
   */
  async diff(): Promise<Change[]>;
}
```

## The complete v2.1 Cache Interface

```ts
interface Cache {
  version: '2';

  // Cache Management
  // ================

  put<T>(doc: StructuredDocument<T>): ResourceDocument;
  patch(op: Operation): void;
  mutate(mutation: Mutation): void;

  peek(identifier: StableRecordIdentifier): ResourceBlob | null;
  peek(identifier: StableDocumentIdentifier): ResourceDocument | null;

  peekRequest(identifier: StableDocumentIdentifier): StructuredDocument<ResourceDocument> | null;

  upsert(
    identifier: StableRecordIdentifier,
    data: ResourceBlob,
    hasRecord: boolean
  ): void | string[];

  // Cache Forking Support
  // =====================

  async fork(): Promise<Cache>;
  async merge(cache: Cache): void;
  async diff(): Promise<object[]>;

  // SSR Support
  // ===========

  async dump(): Promise<ReadableStream<unknown>>;
  async hydrate(stream: ReadableStream<unknown>): void;

  // Resource Support
  // ================

  willCommit(
    identifier: StableRecordIdentifier
  ): void;

  didCommit<T>(
    identifier: StableRecordIdentifier,
    data: StructuredDataDocument<T>
  ): void;

  commitWasRejected(
    identifier: StableRecordIdentifier,
    errors?: StructuredErrorDocument
  ): void;

  getErrors(
    identifier: StableRecordIdentifier
  ): ValidationError[];

  getAttr(
    identifier: StableRecordIdentifier,
    field: string
  ): unknown;

  setAttr(
    identifier: StableRecordIdentifier,
    field: string,
    value: unknown
  ): void;

  changedAttrs(
    identifier: StableRecordIdentifier
  ): Record<string, [unknown, unknown]>;

  hasChangedAttrs(
    identifier: StableRecordIdentifier
  ): boolean;

  rollbackAttrs(
    identifier: StableRecordIdentifier
  ): string[];

  getRelationship(
    identifier: StableRecordIdentifier,
    field: string
  ): Relationship;

  unloadRecord(
    identifier: StableRecordIdentifier
  ): void;

  isEmpty(
    identifier: StableRecordIdentifier
  ): boolean;

  clientDidCreate(
    identifier: StableRecordIdentifier,
    createArgs?: Record<string, unknown>
  ): Record<string, unknown>;

  isNew(
    identifier: StableRecordIdentifier
  ): boolean;

  setIsDeleted(
    identifier: StableRecordIdentifier,
    isDeleted: boolean
  ): void;

  isDeleted(
    identifier: StableRecordIdentifier
  ): boolean;

  isDeletionCommitted(
    identifier: StableRecordIdentifier
  ): boolean;
}
```

## Typescript Support

All implementable interfaces involved in this RFC will be made available via a new package
`@ember-data/experimental-preview-types`. These types should be considered unstable. When 
we no longer consider these types experimental we will mark their stability by migrating
them to `@ember-data/types`.

Documentation for these types is likely to ship
 publicly before the types themselves become installable, and will do so using the final 
 package name (`@ember-data/type`) so that the interfaces are easily explorable on 
 `api.emberjs.com` even before they are mature enough for consumption.

The specific reason for this instability is the need to flesh out and implement an official
pattern for *registries* for record types and their fields. For instance, we expect to change
from `type: string` to the more narrowable and restricted `keyof ModelRegistry & string` when that occurs.

## How we teach this

- updated learning URLs
- updated learning materials (see [emberjs/data#8394](https://github.com/emberjs/data/issues/8394))
- while this adds to cache, it does not add
  the APIs needed for network/store etc.
  future RFCs for exposing these capabilities
  will better define the learning story for the
  average user. Basic integration was defined by [RFC#860](https://github.com/emberjs/rfcs/pull/860)

## Drawbacks

- we haven't explored with implementations on some of these
  ideas; however, it is difficult to explore sans-accepted-rfc
  due to useful features requiring some minimal handshake agreement
  with other layers of the system.
- but singleton + versioning + manager should keep us safely constrained to allow this     
  exploration while still staying within bounds of steerable patterns.

## Alternatives

- a separate document cache
- ephemeral records / buffered proxies
- multiple stores fully isolated (no cache inheritance)
- SSR support at the "source" (Adapter) level only, similar to Shoebox. This approach has a large number of negative performance ramifications.
