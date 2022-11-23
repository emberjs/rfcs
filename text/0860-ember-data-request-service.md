---
stage: accepted
start-date: 2023-11-10
release-date: 
release-versions:
teams:
  - data
prs:
  accepted: https://github.com/emberjs/rfcs/pull/860
project-link:
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
-->

# EmberData | Request Service

## Summary

Proposes a simple abstraction over `fetch` to enable easy management of request/response
flows. Provides associated utilities to assist in migrating to this new abstraction. Adds
 new APIs to the Store to make use of this abstraction.

## Motivation

- `Serializer` lacks the context necessary to serialize/normalize data on a per-request basis
  - This is especially true when performing "actions", RPC style requests, "partial" save
    requests, and "transactional" saves
  - Often users end up needing to pre-normalize in the adapter in order to supply information
    contained in either `headers` or to convert into `JSON` from other forms (such as `jsonb`, `json5` `protocol buffers` or similar)
- `Adapter` is inflexible and difficult to grow as an interface for managing data 
fulfillment from a source.
- Applications have need of a low-level primitive solution for managed fetch to ensure proper headers, authentication, error handling, SSR support, test-waiter support, and request de-duping/caching.
- The `Adapter` pattern stands in the way of pagination-by-default and query caching
- The `Adapter` pattern does not fit with many common data-fetching paradigms today
- The `Adapter` pattern does not fit with transactional saves

## Detailed design

### RequestManager

A `RequestManager` provides a request/response flow in which
configured handlers are successively given the opportunity
to handle, modify, or pass-along a request.

```ts
interface RequestManager {
  request<T>(req: RequestInfo): Future<T>;
}
```


For example:

```ts
import { RequestManager } from '@ember-data/request';
import { Fetch } from '@ember/data/request/fetch';
import Auth from 'ember-simple-auth/ember-data-handler';
import Config from './config';

const { apiUrl } = Config;

// ... create manager
const manager = new RequestManager();
manager.use([Auth, Fetch]);

// ... execute a request
const response = await manager.request({
  url: `${apiUrl}/users`
});
```

**Futures**

The return value of `manager.request` is a `Future`, which allows
access to limited information about the request while it is still
pending and fulfills with the final state when the request completes.

A `Future` is cancellable via `abort`.

Handlers may optionally expose a ReadableStream to the `Future`
for streaming data; however, when doing so the future should not
resolve until the response stream is fully read.

```ts
interface Future<T> extends Promise<StructuredDocument<T>> {
  abort(): void;

  async getStream(): ReadableStream | null;
}
```

The `StructuredDocument` interface is the same as is proposed in emberjs/rfcs#854 but is shown here in richer detail.

```ts
interface RequestInfo {
  /**
   * data that a handler should convert into
   * the query (GET) or body (POST)
   */
  data?: Record<string, unknown>;
  /**
   * options specifically intended for handlers
   * to utilize to process the request
   */
  options?: Record<string, unknown>;
  /**
   * Allows supplying a custom AbortController for
   * the request, if none is supplied one is generated
   * for the request. When calling `next` if none is
   * provided the primary controller for the request
   * is used.
   * 
   * controller will not be passed through onto the immutable
   * request on the context supplied to handlers.
   */
  controller?: AbortController;
  
  // the below options perfectly mirror the
  // native Request interface
  cache?: RequestCache;
  credentials?: RequestCredentials;
  destination?: RequestDestination;
  /**
   * Once a request has been made it becomes immutable, this
   * includes Headers. To modify headers you may copy existing
   * headers using `new Headers([...headers.entries()])`.
   * 
   * Immutable headers instances have an additional method `clone`
   * to allow this to be done swiftly.
   */
  headers?: Headers;
  integrity?: string;
  keepalive?: boolean;
  method?: string;
  mode?: RequestMode;
  redirect?: RequestRedirect;
  referrer?: string;
  referrerPolicy?: ReferrerPolicy;
  /**
   * Typically you should not set this, though you may choose to curry
   * a received signal if calling next. signal will automatically be set
   * to the associated controller's signal if none is supplied.
   */
  signal?: AbortSignal;
  url?: string;
}
interface ResponseInfo {
  headers: Headers;
  ok: boolean;
  redirected: boolean;
  status: number;
  statusText: string;
  type: string;
  url: string;
}

interface StructuredDataDocument<T> {
  request: RequestInfo;
  response: ResponseInfo;
  data: T;
}
interface StructuredErrorDocument extends Error {
  request: RequestInfo;
  response: ResponseInfo;
  error: string | object;
}
type StructuredDocument<T> = StructuredDataDocument<T> | StructuredErrorDocument;
```

A `Future` resolves with a StructuredDataDocument or rejects with a StructuredErrorDocument.

**Request Handlers**

Requests are fulfilled by handlers. A handler receives the request context
as well as a `next` function with which to pass along a request to the next
handler if it so chooses.

If a handler calls `next`, it receives a `Future` which fuulfills to a `StructuredDocument`
that it can then compose how it sees fit with its own response.

```ts
type NextFn = <P>(req: RequestInfo) => Future<P>;

interface Handler {
  request<T>(context: RequestContext, next: NextFn): T;
}
```

`RequestContext` contains information about the request as well as a few methods
 for building up the `StructuredDocument` and `Future` that will be part of the
 response.

```ts
interface RequestContext<T> {
  readonly request: RequestInfo;

  setStream(stream: ReadableStream | Promise<ReadableStream | null>): void;
  setResponse(response: ResponseInfo | Response | null): void;
}
```

A basic `fetch` handler with support for streaming content updates while
the download is still underway might look like the following, where we use
[`response.clone()`](https://developer.mozilla.org/en-US/docs/Web/API/Response/clone) to `tee` the `ReadableStream` into two streams.

A more efficient handler might read from the response stream, building up the 
response data before passing along the chunk downstream.

```ts
const FetchHandler = {
  async request(context) {
    const response = await fetch(context.request);
    context.setResponse(reponse);
    context.setStream(response.clone().body);

    return response.json();
  }
}
```

Request handlers are registered by configuring the manager via `use`

```ts
manager.use([Handler1, Handler2])
```

Handlers will be invoked in the order they are registered ("fifo", first-in 
first-out), and may only be registered up until the first request is made. It
is recommended but not required to register all handlers at one time in order
to ensure explicitly visible handler ordering.

**Stream Currying**

`RequestManager.request` differs from `fetch` in one **extremely crucial detail**
and we feel the need to deeply describe how and why.

For context, it helps to understand a few of the use-cases that RequestManager
is intended to allow.

- Historically EmberData could not be used to manage and return streaming content (such as
 video files). With this change, it can be. (The Identifiers RFC and Cache 2.1 RFCs also 
 make this ability pervasive throughout all layers of EmberData)
- It might be the case that a handler "tees" or "forks" a request, fulfilling it by either
  making multiple parallel `fetch` requests, or by calling `next` multiple times, or by
  fulfilling part of the request from one source (one API, in-memory, localStorage, IndexedDB
   etc.) and the rest from another source (a different API, a WebWorker, etc.)
- Handlers may only be amending the request and passing it along, for instance an Auth handler
 may simply be ensuring the correct tokens or headers or cookies are attached.

`await fetch(<req>)` behaves differently than some realize. The fetch promise resolves not
once the entirety of the request has been received, but rather at the moment headers are 
received. This allows for the body of the request to be processed as a stream by application
code *while chunks are still being received by the browser*. When an app chooses to
`await response.json()` what actually occurs is the browser reads the stream to completion
and then returns the result. Additionally, this stream may only be read **once**.

In designing the `RequestManager`, we do not want to eliminate this ability to subscribe to
and utilize the stream by either the application or the handler. We believe it crucial that
the full power and flexibility of native APIs remains in developers hands, and do not want
to create a restriction such that developers feel the need to create complicated workarounds
for what would feel like an unnecessary restriction to gain access to built-in APIs.

However, because there is potentially a chain of handlers involved, and because there are 
potentially multiple streams involved, and because we require that `await manager.request(<req>)`
 resolves with fully realized content, we find ourselves in a design conundrum.

We have considered several variations on how to support streams: from a two-tiered promise structure
 similar to `fetch` (which quickly fails due to the chained nature of handlers), to enforcing that 
handlers synchronously call `setStream` either with a ReadableStream or a promise resolving to one.

Each variation has had drawbacks, some were critical and some simply had poor ergonomics. What we
have arrived at is this:

Each handler may call `setStream` only once, but may do so *at any time* until the promise that
the handler returns has resolved. The associated promise returned by calling `future.getStream`
will resolve with the stream set by `setStream` if that method is called, or `null` if that method
has not been called by the time that the handler's request method has resolved.

Handlers that do not create a stream of their own, but which call `next`, should defensively
pipe the stream forward. While this is not required (see automatic currying below) it is better
to do so in most cases as otherwise the stream may not become available to downstream handlers
or the application until the upstream handler has fully read it.

```ts
context.setStream(future.getStream());
```

Handlers that either call `next` multiple times or otherwise have reason to create multiple 
fetch requests should either choose to return no stream, meaningfully combine the streams,
or select a single prioritized stream.

Of course, any handler may choose to read and handle the stream, and return either no stream or
a different stream in the process.

**Automatic Currying of Stream and Response**

In order to simplify what we believe will be a common case for handlers which are merely decorating
a request, if `next` is called only a single time and `setResponse` was never called by the handler
the response set by the next handler in the chain will be applied to that handler's outcome. For 
instance, this makes the following pattern work `return (await next(<req>)).data;`.

Similarly, if `next` is called only a single time and neither `setStream` nor `getStream` was
 called, we automatically curry the stream from the future returned by `next` onto the future returned by the handler.

Finally, if the return value of a handler is a `Future`, we curry the entire thing. This makes the
 following possible and ensures even `data` and `error` is curried when doing so: `return next(<req>)`.

In the case of the `Future` being returned from a handler not using `async/await`, `Stream` proxying is automatic and immediate and does not wait for the `Future` to resolve. If the handler uses `async/await` we have no ability to detect the Future until the handler has fully resolved. This means that if using `async/await` in your handler you should always pro-actively pipe the stream.

**Using as a Service**

Most applications will desire to have a single `RequestManager` instance, which
can be achieved using module-state patterns for singletons, or for Ember 
applications by exporting the manager as an Ember service.

*services/request.ts*
```ts
import { RequestManager } from '@ember-data/request';
import { Fetch } from '@ember/data/request/fetch';
import Auth from 'ember-simple-auth/ember-data-handler';

export default class extends RequestManager {
  constructor(createArgs) {
    super(createArgs);
    this.use([Auth, Fetch]);
  }
}
```

**Using with the Store Service**

Assuming a manager has been registered as the `request` service.

*services/store.ts*
```ts
import Store from '@ember-data/store';
import { service } from '@ember/service';

export default class extends Store {
  @service('request') requestManager;
}
```

Alternatively to have a request service unique to the store:

```ts
import Store from '@ember-data/store';
import { RequestManager } from '@ember-data/request';
import { Fetch } from '@ember/data/request/fetch';

export default class extends Store {
  requestManager = new RequestManager();

  constructor(args) {
    super(args);
    this.requestManager.use([Fetch]);
  }
}
```

If using the package `ember-data`, the following configuration will automatically be done in order
to preserve the legacy Adapter and Serializer behavior. Additional handlers or a service injection
like the above would need to be done by the consuming application in order to make broader use of
`RequestManager`.

```ts
import Store from '@ember-data/store';
import { RequestManager } from '@ember-data/request';
import { LegacyHandler } from '@ember-data/legacy-network-handler';

export default class extends Store {
  requestManager = new RequestManager();

  constructor(args) {
    super(args);
    this.requestManager.use([LegacyHandler]);
  }
}
```

### Using `store.request(<req>)`

The `Store` will add support for using the `RequestManager` via `store.request(<req>)`.

```ts
class Store {
  request<T>(req: RequestInfo): Future<Reified<T>>;
}
```

There are three significant differences when calling `store.request` instead of `requestManager.request`.

1) the Store will be added to `RequestInfo`, and an additional `cacheOptions` property is available

```ts
interface StoreRequestInfo extends RequestInfo {
  cacheOptions?: { key?: string, reload?: boolean, backgroundReload?: boolean };
  store: Store;
}
```

2) The `StructuredDocument` is supplied to `cache.put(doc)` and the return value's
 `data` member is altered to either a single record or array of records resulting
  from instantiating the entities contained in the `ResourceDocument` returned by
  `cache.put`.

3) Both an operation (`op`) and and array of identifiers (`records`) may be provided
  as part of the request. While this information could also be included in `options`,
  we are giving it top-level precedence since handlers which perform data normalization
  will almost always require this information.

  `op` may be any `string` that your handlers will recognize, though EmberData will provide
   an `op` matching one of the current Adapter request types when it is used to build the
   RequestInfo object.

   `records` should be all records expected to be saved or fetched during the course of
   the request. Similarly, EmberData will populate this for you when using the request-builders
   or when the request is generated by the Store. This list will be used to update the status
   of the `RequestStateService` detailed in [RFC #466](https://rfcs.emberjs.com/id/0466-request-state-service)

```ts
interface StoreRequestInfo extends RequestInfo {
  cacheOptions?: { key?: string, reload?: boolean, backgroundReload?: boolean };
  store: Store;

  op?: 'findRecord' | 'updateRecord' | 'query' | 'queryRecord' | 'findBelongsTo' | 'findHasMany' | 'createRecord' | 'deleteRecord';
  records?: StableRecordIdentifier[];
}
```

**Background Reload Error Handling**

When an error occurs during a background request we will update the cache with the StructuredErrorDocument but will swallowed the Error at that point.

This prevents consuming applications from being required to catch the error unless
they wish to via a handler.

### RequestStateService

We do not intend to make any adjustments to the RequestStateService at this time, though
this new paradigm enables us to easily manage a list of requests key'd by URL that could
be useful for both application code and the Ember Inspector. If you are interested in adding
such support, we would accept an RFC. With the greatly improved flow this RFC brings we
expect that the overall design of the RequestStateService ought to be revisited.

### Cache Lifetimes

In the past, cache lifetimes for single resources were controlled by either 
supplying the `reload` and `backgroundReload` options or by the Adapter's hooks
for `shouldReloadRecord`, `shouldReloadAll`, `shouldBackgroundReloadRecord` and 
`shouldBackgroundReloadAll`.

This behavior will now be controlled by the combination of either supplying `cacheOptions`
on the associated `RequestInfo` or by supplying a `lifetimes` service to the `Store`.

```ts
class Store {
  lifetimes: LifetimesService;
}

interface LifetimesService {
  isExpired(url: string, method: HTTPMethod) {}
}
```

**Legacy Compatibility**

In order to support the legacy adapter-driven lifetime behaviors of `findRecord`
and similar store methods, these methods will still consult the adapter prior to
consulting the lifetimes service. Requests that originate through `store.request`
will not consult the Adapter methods.

### Legacy Adapter/Serializer Support

In order to provide migration support for Adapter and Serializer, a `LegacyNetworkHandler` would be
provided. This handler would take a request and convert it into the older form, calling the appropriate
Adapter and Serializer methods. If no adapter exists for the type (including no application adapter), this
handler would call `next`. In this manner an app can incrementally migrate request-handling to this
new paradigm on a per-type basis as desired.

The package `ember-data` would automatically configure this handler. If not using `ember-data`
this configuration would need to be done explicitly.

We intend to support this handler through at least the 5.x series, not deprecating it's usage
before 6.0.

Similarly, the methods `adapterFor` and `serializerFor` will not be deprecated until at least 6.0;
however, it should no longer be assumed that an application has an adapter or serializer at all.

### Migrating Away From Legacy Finders

In order to support transitioning to this new paradigm, we would introduce new url-building
and request-building utility functions in a new package (`@ember-data/request-utils`) that
closely mirror what occurs by using the corresponding store and Adapter methods today.

Note: the lack of `findAll` in this list is intentional, we do not intend to implement this
separately from `query`.

```ts
import { findRecord, queryRecord, query, updateRecord, createRecord, deleteRecord, saveRecord } from '@ember-data/request-utils';

const { data: user } = await store.request(findRecord('user', '1'));
const { data: user } = await store.request(queryRecord('user', { username: 'runspired' }));
const { data: users } = await store.request(query('user', {}));

await store.request(updateRecord(user));
await store.request(createRecord(user));
await store.request(deleteRecord(user));
await store.request(saveRecord(user));
```

Each of these request-builders returns an object satisfying the `RequestInfo` interface, which
could also be manually constructed.

Additionally, a url-builder similar in behavior to the `BuildURLMixin` is provided.
Notable differences include that is also serializes query params into the URL, and
assumes the first argument is the "path for type".

The following config properties will be supply-able via the app's `ember-cli-build`

```ts
interface Config {
  '@embroider/macros': {
    setConfig: {
      '@ember-data/request-utils': {
        {
          apiNamespace: string;
          apiHost: string;
        }
      }
    }
  }
}
```


```ts
import { buildUrl } from '@ember-data/request-utils';

// findRecord with include
const url = buildUrl('user', '1', { include: 'friends' });

// query page 3 of users
const url = buildUrl('users', null, { limit: 25, offset: 50 });

// query for a single user
const url = buildUrl('user', null, { username: 'runspired' });

// query the first page of comments for post 1
const url = buildUrl('post/1/comments/list', null, { limit: 10, offset: 0 });
```

### Deprecating Legacy Finders

We would not immediately deprecate methods on the Store for requesting data until at
least 6.0; however, applications should begin migrating all requests to this new
paradigm and expect that the following methods will be deprecated at some point during
the 6.x cycle

  - `store.findRecord`
  - `store.findAll`
  - `store.query`
  - `store.queryRecord`
  - `store.saveRecord`

Users that want to maintain these finder methods for longer would be able to add them back
 within their own application or library if desired; however, because these methods cannot
 easily utilize the full feature set of the cache, pagination, or request-manager we expect
 that their utility will diminish quickly.

### Migrating Away from Serializers

We do not intend to provide a direct replacement of Serializers in any form. Instead,
given the current power and flexibility of the Cache, we recommend aligning the
Cache implementation with your API implementation.

If data normalization is still needed, we recommend writing a few helper functions
that a handler can use to quickly transform the data as necessary. Due to having better
context of the request, and due to the much smaller surface area to reason about, writing
a function to transform data between formats should prove to be simple, quick and effective.
We expect some addons may be created that offer helper functions for common transformations.

Since Serializers will not be officially deprecated until some point after 6.0, we feel
that this is more than ample time for applications and addons to explore this space and
either become comfortable with the realization that such data transformation is largely
 unecessary and wasteful or can be done via much simpler and surgical mechanisms.

Of course, users can always choose to continue using Serializers (and Adapters) forever.
Their deprecation within EmberData will be scoped to (1) EmberData itself no longer being
aware of the concept and (2) the `adapter` and `serializer` packages being deprecated.

If desired, other libraries could take on support of these packages, and make use of
public APIs to restore these behaviors, utilizing the same public APIs EmberData will
use to support them until deprecated. We suspect, however, the insane improvement in ergonomics
and feature-set that this shift brings will –over the course of the few years prior to full
removal– prove to users that the Adapter and Serializer world is no longer the best paradigm
for their applications.

### Typescript Support

Although EmberData has not more broadly shipped support for Typescript, experimental types
will be shipped specifically for the RequestManager package. We can do this because the lack
of entanglement with the other packages affords us the ability to more safely ship this subset
of types while the others are still incomplete.

Types for other packages will eventually be provided but we will not rush them at this time.

## How we teach this

- EmberData should create new documentation and guides to cover using the RequestManager.

- Existing documentation and guides should be re-written to either reference these new
patterns or to be clear that they discuss a legacy pattern that is no longer recommended.

- The learning story for EmberData should be reworked to one that incrementally grows from
  a simple abstraction over fetch, to fetch with a cache, to fetch with a cache and resource
  graph and presentational concerns.

## Drawbacks

Historically, Adapters hid away construction of requests from app-developers which kept 
application code focused only on working with data that was magically fetched and processed
in the background.
 
When this worked, it worked very well, and many users have loved this magic deeply. However,
this abstraction came at the great cost of making EmberData difficult to fit into many common
scenarios, difficult to reason about and debug when data-fetching failed, and difficult to
extend when even very trivial changes to request construction were required.
 
We do not feel the occassional magic of it all working outweighs the drawbacks of
keeping the system as is, and so we have chosen a slightly more verbose approach that grants
developers flexibity, power, and ease-of-use.

## Alternatives

We considered building this over the existing Adapter interface, deprecating Serializers and
instead encouraging data-transformation to be done within the Adapter. In fact, this pattern
is fully possible today, we could just better document it and do nothing more. However, this
approach does not solve the need for more general request management, nor does it interact
well with common development paradigms such as GraphQL query building, nor does it allow us
to introduce pagination-by-default.