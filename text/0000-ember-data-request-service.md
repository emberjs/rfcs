---
stage: accepted
start-date: 
release-date: 
release-versions:
teams:
  - data
prs:
  accepted: # update this to the PR that you propose your RFC in
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
- `Adapter` is inflexible and difficult to grow as an interface for managing data 
fulfillment from a source.
- Applications have need of a low-level primitive solution for managed fetch to ensuring proper headers, authentication, error handling, SSR support, test-waiter support, and request de-duping/caching.

## Detailed design

#### RequestManager

A `RequestManager` provides a request/response flow in which
configured middlewares are successively given the opportunity
to handle, modify, or pass-along a request.

```ts
import RequestManager, { Fetch } from '@ember-data/request';
import Auth from 'ember-simple-auth/ember-data-middleware';
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

Middlewares may optionally expose a ReadableStream to the `Future`
for streaming data; however, when doing so the future should not
resolve until the response stream is fully read.

```ts
interface Future<T> extends Promise<StructuredDocument<T>> {
  abort(): void;

  async stream(): ReadableStream | null;
}
```

The `StructuredDocument` interface is the same as is proposed in RFC 854 and copied below:

```ts
interface StructuredDocument<T> {
  request: {
    url: string;
    cache?: { key?: string, reload?: boolean, backgroundReload?: boolean };
    method: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
    data?: Record<string, unknown>;
    options?: Record<string, unknown>;
    headers: Record<string, string>;
  }
  response: {
    status: HTTPStatusCode;
    headers: Record<string, string>;
  }
  data: T;
  error?: Error;
}
```

**Registering Middlewares**

**The Middleware API**

**Using as a Service**

Most applications will desire to have a single `RequestManager` instance, which
can be achieved using module-state patterns for singletons, or for Ember 
applications by exporting the manager as an Ember service.

*services/request.ts*
```ts
import RequestManager, { Fetch } from '@ember-data/request';
import Auth from 'ember-simple-auth/ember-data-middleware';

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
import RequestManager, { Fetch } from '@ember-data/request';

export default class extends Store {
  requestManager = new RequestManager();

  constructor(args) {
    super(args);
    this.requestManager.use([Fetch]);
  }
}
```



We would introduce new url-building and request building utility functions in a new package
`@ember-data/request-utils`.

We would introduce a new middleware in it's own package `@ember-data/legacy-network-middleware`
to encapsulate legacy behaviors of adapters, and serializers.

We would introduce a new lifetimes behavior to the `Store` to replace the reload APIs on `Adapter`.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
