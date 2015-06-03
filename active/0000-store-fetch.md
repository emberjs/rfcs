- Start Date: (fill me in with today's date, 2015-06-03)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes a new method on the Ember Data store for asynchronously retrieving records where if the record is already found in the store Ember Data will return the cached record then in the background ask the adapter to re-retrieve the record to make sure it has the freshest data. If the record is not in the store Ember Data will make a request like normal and it will not resolve the returned promise until the data is returned from the adapter.

# Motivation

## Use Cases

When it comes to fetching records, there are several different behaviors
that users may expect. The behavior that users expect is influenced by
unique quirks in their data model, pre-existing expectations based on
traditional development models, and implementation details of their
adapter.

Fundamentally, users may expect or want one of the following sets of
behavior when fetching a model for the `model()` hook:

* Fetching data from the server the first time a record is requested,
  but using only cached data subsequent times the route is entered.
  (This is the current behavior of `find()`.)
* Fetching new data every time the route is entered. The route will
  "block" (show a loading spinner) until fresh data is received.
* Using local data if available, but otherwise not triggering any
  fetches if the data is not available. This is useful if records will
  be pushed into the store ahead of time, e.g. by a socket, and
  non-existence in the store means non-existence on the server.
* Immediately returning a model with local data if available, rendering
  the route's template immediately, and updating the record in the
  background. If the record changes after conferring with the server,
  the template is re-rendered.

## Discussion

### Fetch Only On Initial Render

In this model, the record data is fetched from the server the first time
a record with a given ID is requested. Subsequent requests (e.g. leaving
and re-entering the same route) use locally cached data. This is the
strategy used by the current `find()` method.

The advantage of this model is that it keeps conferring with the server
to a minimum. Once data is loaded, the client can render new routes with
the model data that it has cached, without a network roundtrip.

Additionally, in some data models, records are immutable. For example, on
Twitter, tweets never change. In an email app, emails cannot change once
they are sent. Asking the server for the most up-to-date version of an
immutable record is a waste of resources.

The downsides of this model are two-fold.

First, this model is surprising to new developers. When navigating
between pages, they expect the most up-to-date representation to be
fetched and displayed every time.

Second, even for developers who understand what is happening, it is very
easy for long-lived applications to accumulate stale information,
particularly if the model they are displaying updates frequently.
Developers must somehow disambiguate between the first time a model is
looked up, and allowing it to proceed, and detecting when a cached model
is being used and updating it manually.

### New Fetch Every Render

The advantages of this model are that it most closely matches the mental
model for developers coming from server-rendered and jQuery backgrounds.
In that model, every time a new page is loaded, the most up-to-date
information is guaranteed to be displayed. Because each page navigation
triggers a fetch from the database, the only way for information to
become stale is for the user to stop navigating.

The downside of this model is that it eliminates many of the advantages
of client-side routing. In traditional client apps, data is stored
locally, and navigations use that local data. In this model, every page
transition is blocked awaiting a network response from the server. It's
a slight improvement in that the data should be much smaller than a full
HTML page, but it is often latency and not bandwidth that causes
slowdowns.

### Never Fetch

While an edge case, many Ember Data users have requested the ability to
fetch a record out of the store only if it exists locally.

One use case is for stores that are optimistically populated via pushed
data from a socket. In that case, if the record doesn't exist in the
store, it means that it doesn't exist on the server.

For obvious reasons, this is an uncommon case for the majority of apps.
While we should support it, it should not be part of the happy path for
new developers.

### Immediate Render, Background Refresh

In this model, the first time a record is requested, it blocks the
render and shows a loading spinner. On subsequent requests, the locally
cached data is displayed and the render happens immediately without
making the user wait.

However, in the background, the store also kicks off a request to the
adapter to update the record. When the new data comes in, the record is
updated, and if there have been changes to the record since the initial
render, the template is re-rendered with the new information.

This is the model that I believe strikes the best tradeoff among the
options available.

First, it preserves the speed of client-side navigation. Once data for a
record is cached, transitioning to any route that relies on it is nearly
instantaneous and has no network bottleneck.

Second, because it triggers a background update, even users who expect a
new fetch every time will not be surprised as, ideally after a few
milliseconds, the new data will arrive and be persisted into the DOM.

Third, in most applications, models are not changing frequently.  Most
of the time, the cached version in the Ember Data store will be
identical to the latest server revision. In those cases, there is no
point in making users stare at a loading spinner

Of course, there are several downsides to this model that we should keep
in mind. For immutable records, fetching a new version in the background
is wasteful of bandwidth and server capacity and we should allow
developers to opt out of this behavior.

A second related case is apps using a socket to subscribe to record
changes once a record is fetched. In those cases, fetching up-to-date
information on subsequent requests for the model is wasteful because
they have guaranteed that they will keep the model up-to-date via change
events from the server. In this case, we need a way for adapter authors
to signal that subsequent update requests for a record are a no-op.

Third, it may be an unpleasant user experience for new information to
pop in suddenly after the initial render, particularly for records that
frequently change in dramatic ways. In those instances, we should make
sure we give developers the tools to build UIs that can indicate to the
user that the information is being updated, perhaps by greying it out or
displaying a loading spinner.

# Detailed design

## Proposal

### Fetch

Going forward, the recommended API for fetching a record in Ember Data
should be:

```js
this.store.fetch('person', 1);
```

This will return a promise that:

1. Waits to resolve until the data is fetched from the server, on the
   initial request.
2. Resolves immediately with the locally cached request for subsequent
   requests, but triggers a request to the server for the updated
   version and updates the record in-place if there are changes.

For adapters that use a socket to keep records up-to-date, this
`fetch()` should also cause the socket to subscribe to change events for
this record. Subsequent requests should be a no-op since those events
will keep the record up-to-date.

Whether pushed events or XHR requests are used to keep the record
up-to-date is an adapter implementation detail. The fundamental
guarantee of `fetch()` is:

> Give me the information you have available locally, then give me the
> most up-to-date information as soon as possible.

In sockets, that means that the locally available information is _also_
the most up-to-date information and no further server round-trips are
necessary.

To tell the fetch to not resolve the promise until the most up-to-date
information is fetched (i.e., to implement the _New Fetch Every Render_
approach described above), you can pass a flag that indicates that the
promise should not be resolved until the adapter has provided refreshed
information:

```js
this.store.fetch('person', 1, { waitForUpdate: true });
```

Currently, the `find()` method takes an optional third parameters that
is passed to the adapter. In this API, that data structure is moved to
a field in the new options hash:

```js
this.store.fetch('person', 1, {
  waitForUpdate: true,
  adapterOptions: { comment_id: 1 }
});
```

### `isUpdating` Flag

To assist developers in building UIs that communicate the state of
models to their users, we should provide a helper that allows developers
to show UI elements when a model is in the process of being updated via
`fetch()`.

I propose adding an `isUpdating` flag to models, which can be used to
conditionally show a spinner:

```handlebars
<h1>{{post.title}}</h1>
{{#if isUpdating post}}
  <small>Updating...</small>
{{/if}}

<p>{{post.body}}</p>
```

(Currently, only `RecordArray`s have an `isUpdating` flag.)

Models have an `isReloading` flag. This will be deprecated in favor of the new `isUpdating` flag.

# Drawbacks

Why should we *not* do this?

After the record has been updated in the background Ember's Data binding will cause any views to automatically update with the latest changes. This can result an a surprising "popping" effect which is especially pronounced when the background fetch resolves quickly (The user sees an initial render with the stale data then a quick re-render with the fresh data).  




# Alternatives

What other designs have been considered? What is the impact of not doing this?

One alternate option could be for Ember Data to track an expires token on a model. This would allow Ember 
Data to behave like a caching proxy when fetching. If the record is expired, fetch should block. 
If the record is not expired it would return a resolve the record right away however still issue a
second request. 

When used with backends that do not return an expires token. Ember Data would assume that the 
record is stale (this could be configured on the adapter).


# Unresolved questions

The term `fetch` is already used by Ember Data for the behavior of always finding a record from the adapter. Alternately, if we ever decided we wanted to split `fetch` into 2 methods for the single record and multiple record cases we would run into the existing `fetchById`/`fetchAll` methods. 


