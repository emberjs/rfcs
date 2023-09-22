---
stage: accepted
start-date: 2023-09-18T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - data
prs:
  accepted: https://github.com/emberjs/rfcs/pull/964
project-link:
suite: 
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
suite: Leave as is
-->

# EmberData | Deprecate Legacy Request Support

## Summary

Deprecates methods on `store` and `model` that utilize non-request-manager request paradigms. These
methods are no longer recommended in the face of the greater utility of `store.request`.

Deprecates methods on store associated to older request paradigms / the inflexibility of older paradigms.

These deprecations would target 6.0.

## Motivation

This RFC is a debt collection. The newer RequestManager paradigm offers a pipeline approach to requests and preserves the context of the request throughout its lifecycle. This newer paradigm solves the issues
with limited power and flexibility that the adapter/serializer approach suffered and which led to so many ejects to fetch+push, work arounds, frustrations, and library removals.

The RequestManager paradigm ensures that any request starts by providing FetchOptions. These may be built programmatically (for instance for relationships or collections that return links information from the API), or by builders, or even manually.

Users have the choice of providing the body for a request at the point of the request, or inserting one later using a request handler. For both cases, EmberData provides access to the cache and its APIs for diffing state changes, as well as `serializePatch` and `serializeResources` utils for each cache implementation that we ship.

The paradigm has the simple goal of "use the platform". Importantly, it is this usage that allows us to intelligently cache requests and documents alongside resources, opening up vast possibilities for helpful new features for EmberData that these older methods do not support.

The legacy pattern's inflexibility meant that users often needed to eject from using adapter/serializer paradigms to fetch data. The new paradigm does not have these constraints, so we wish to deprecate methods that served only as work arounds or only work with these now legacy concepts.

## Detailed design

### Deprecating Historical Request Methods

Currently, the Store allows users to continue using the historical methods for fetching or mutating data.

**Group 1**

- `store.findRecord`
- `store.findAll`
- `store.query`
- `store.queryRecord`
- `store.saveRecord`
- `model.save`
- `model.reload`
- `model.destroyRecord`

These historical methods include the `hasMany` and `belongsTo` async auto-fetch behaviors on `@ember-data/model`

**Group 2**
- accessing an async belongsTo
- accessing an async hasMany

As well as corresponding relationship reference fetching methods

**Group 3**
- `HasManyReference.load()`
- `HasManyReference.reload()`
- `BelongsToReference.load()`
- `BelongsToReference.reload()`

These historical request methods currently translate their arguments into the shape expected by RequestManager with a few major caveats:

- they require using the LegacyCompatHandler and adapter/serializer logic or something mimicking it
- they are not as flexible at the callsite
- they are not cacheable since they do not set cacheKey and do not provide a url

Now that builders have shipped in 5.3, deprecating all of group 1 allows us to begin simplifying the mental model of how EmberData should be used.

Groups 2 and 3 should not be deprecated until we've either provided an alternative decorator to replace async `belongsTo` and `hasMany` or shipped [SchemaRecord](#8845)

#### What to do instead

Examples here are shown for apps that use `JSON:API`. Apps using other paradigms should use the builders for `REST` or `ActiveRecord` if
applicable, or author their own (or a new community lib!) if not.

- `store.findRecord`
- `model.reload`

```ts
import { findRecord } from '@ember-data/json-api/request';

const result = await store.request(findRecord('user', '1'));
const user = result.content.data;
```

- `store.findAll`

If you don't want all records in the store + what the API says

```ts
import { query } from '@ember-data/json-api/request';

const result = await store.request(query('user'));
const users = result.content.data;
```

If you do want all records in the store + what the API says. Note,
we would heavily discourage this approach having watched as it leads
to difficult to disentangle complexity in applications.

```ts
import { query } from '@ember-data/json-api/request';

await store.request(query('user'));
const users = store.peekAll('user');
```

- `store.query`

For requests that are expected to send a "body" to the API see notes
in the saveRecord section below.

```ts
import { query } from '@ember-data/json-api/request';

const result = await store.request(query('user', params));
const users = result.content.data;
```

- `store.queryRecord`

```ts
import { query } from '@ember-data/json-api/request';

const result = await store.request(query('user', { ...params, limit: 1 } ));
const user = result.content.data[0] ?? null;
```

- `store.saveRecord`
- `model.save`

For requests that are expected to send a "body" to the API applications
may choose to either serialize the body at the point of the request or
to implement a Handler for the RequestManager to do so.

EmberData does not provide a default handler which serializes because this
is a unique concern for every app. However, EmberData does provide utilities
on both the Cache and for some of the builders to make this easy.

For JSON:API we show the "at point of request" approach using the utils
provided by the `@ember-data/json-api` package here.

**for create**

```ts
import { recordIdentifierFor } from '@ember-data/store';
import { createRecord, serializeResources } from '@ember-data/json-api/request';

const record = store.createRecord('user', {});
const request = createRecord(record);
request.body = JSON.stringify(
  serializeResources(
    store.cache,
    recordIdentifierFor(record)
  )
);

await store.request(request);
```

**For update**

```ts
import { recordIdentifierFor } from '@ember-data/store';
import { updateRecord, serializePatch } from '@ember-data/json-api/request';

user.name = 'Chris';

const request = updateRecord(user);
request.body = JSON.stringify(
  serializePatch(
    store.cache,
    recordIdentifierFor(user)
  )
);

await store.request(request);
```

Note if you only wanted to save the single mutation you just made, you could.

```ts
import { updateRecord, serializePatch } from '@ember-data/json-api/request';

// local mutation (reflected on model immediately)
user.name = 'Chris';

const request = updateRecord(user);
request.body = JSON.stringify(
  {
    data: {
      type: 'user',
      id: user.id,
      attributes: {
        name: user.name
      }
    }
  }
);

await store.request(request);
```

**for delete**

- also `model.destroyRecord`

```ts
import { deleteRecord } from '@ember-data/json-api/request';

store.deleteRecord(user);
await store.request(deleteRecord(user));
store.unloadRecord(user);
```


### Deprecating Store Data Munging

Additionally, we deprecate store methods for data munging:

- `store.pushPayload`
- `store.serializeRecord`

#### What to do instead 

**For Modern Apps**

- align the cache and API format, use `store.push` to upsert
- use the same normalization functions written for handling responses in the app's request handlers, use `store.push` to upsert
- migrate the request to just use `RequestManager` now that the limitations of the adapter pattern are gone

**For Apps still using Legacy**

- Use `store.serializerFor` and `serializer.normalizeResponse` to normalize payloads before using `store.push`.
- Some previous discussion https://github.com/emberjs/data/issues/4213#issuecomment-413370235
- Some examples of how to work without this method: https://github.com/emberjs/data/pull/4110#issuecomment-417391930


## How we teach this

- API Docs, guides, and the tutorial should remove usage examples of older patterns, replacing them with
  newer patterns.

## Drawbacks

The only drawback here is that this deprecation doesn't go further. We do not at this time deprecate adapters
and serializers or the LegacyNetworkHandler. This is because to do so we must also deprecate the auto-fetch
behaviors of async relationships in `@ember-data/model`. We prefer to deprecate those aspects of the system
only once replacements are firmly in place.

However, we think continuing to clarify the mental model for everything else is important, especially because
`@ember-data/model` is not a required component of an EmberData installation and users can utilize everything
else today without Adapter/Serializer/Model should they so choose.

## Alternatives

Convert all of these APIs to expect builder input. We have not taken this avenue as we feel the scope of that
deprecation would be much harder to manage and much tougher to navigate. However, an application may choose to
extend the store and implement any of the request method they choose in this manner because they have the knowledge
of which builder to use.

```ts
import { findRecord } from 'app/builders';

class extends Store {
  async findRecord(type, id, options?): Promise<Record | null>; {
    const result = await this.request(findRecord(type, id, options));
    return result.content.data;
  }
}
```
