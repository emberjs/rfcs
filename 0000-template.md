---
Stage: Accepted
Start Date: 2021-04-23
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): ember-data
RFC PR: 
---

# EmberData | State Machine Updates

## Summary

Provides additional state-machine substates for the `@ember-data/model` `currentState` property.

## Motivation

Several corner cases today lose necessary information when bucketed into "fit everything" states.
Adding these substates will allow us to retain that information leading to better state management
capabilities both internally and for consuming applications.

## Detailed design

The following substates would be available via `currentState.stateName`. These states would
have additional boolean flag states described below and would inherit the flags of their
parents. For instance, the `root.empty.deleted` substate would inherit `isEmpty = true` from the
empty state, and the `root.loading.empty` substate would inherit `isLoading = true` from the loading 
state. 

**root.empty.deleted**

This substate would be entered when a record is unloaded from either the `created.uncommitted` 
or `deleted.saved` states. When entered `currentState.isDeleted` will be `true`. In practical
effects this helps us determine whether artifacts for an unloaded record can safely be fully
removed from the store, since otherwise the context of the record being empty is lost.

**root.empty.preloaded**

This substate would be entered when a record is initialized via `findRecord` with data provided
via the `preload` option. When entered `currentState.isPreloaded` will be `true`. It is unlikely
this state will be user observable as in almost all cases the record will immediately transition
to `loading.preloaded`, but this state is necessary for that transition to occur correctly. 

**root.loading.empty**

This substate would be entered when a record is in the process of being found by the server but
has not yet ever received a payload nor was preloaded. When entered `isEmpty` will be `true`. In
practical effect this will cause loading records without any data to be filtered from live-arrays
and relationships.

**root.loading.preloaded**

This substate would be entered when a record is initialized via `findRecord` with data provided
via the `preload` option once the request to retrieve the record has begun. When entered 
`currentState.isPreloaded` will be `true`. This allows preloaded records to participate in
record-arrays and relationships they may otherwise be hidden from.

**root.deleted.saved.new**

This substate would be entered when a record created on the client via `store.createRecord` whose
data was never persisted is subsequently removed either by `deleteRecord()`, `record.deleteRecord()`
+ `record.saveRecord()`, or (pending changes to the destroy semantics) `destroyRecord()`. This
state is observable between the originating calls and a call to `unloadRecord()`. When entered 
`isNew` will be `true`. In practical effects this allows us to know that the record never held any
remote state, cannot be resurrected via an API call, and should be hidden from any record-arrays or
relationships.

## How we teach this

`currentState.stateName` is largely an obscure API with little documentation, but the state flags
it produces (`isNew` `isDeleted` `isLoaded` etc.) are public. Since this RFC seeks to ensure that
those flags produce the correct state in these edge cases, largely documentation materials will not
need to be updated as this is effectively a bugfix, and would be entirely so if it were not for the 
observable outcome of new `stateName` values.

## Drawbacks

`@ember-data/model` is a paradigm EmberData is actively designing away from by enabling APIs that
allow for greater exploration in the modeling space (such as `ember-m3`). To that effect we even
have produced [RFC#463 RecordData State](https://emberjs.github.io/rfcs/0463-record-data-state.html),
[RFC#465 RecordData Errors](https://emberjs.github.io/rfcs/0465-record-data-errors.html), and
[RFC#466 RecordData State Service](https://emberjs.github.io/rfcs/0466-request-state-service.html) to 
break apart the conceptsof the state machine to enable alternatives.

However, as we have not yet produced a built-in alternative to `@ember-data/model` and as these substates
help to bring more clarity and correctness to both internal code and user observable behaviors and 
properties it seems better to fix the APIs we do currently ship as the default as this will only aide in
our ability to ship an alternative.

There is likely at least *some* app code usage of checks such as `currentState.stateName === 'root.loading'` and `currentSate.stateName === 'root.deleted.saved'`, however a code search of addons suggests
this is barely utilized and the migration path for apps that hit this is a minimal change in their check
to either check for the flag (preferred), the substate, or for the presence of the parent state in the full path.

## Alternatives

Leave things as they are, or add non-state-machine state flags that we manage separately so as to avoid
observable changes to `currentState.stateName`.
