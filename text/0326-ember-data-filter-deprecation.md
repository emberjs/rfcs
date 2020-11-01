---
Start Date: (fill me in with today's date, 2018-04-18)
RFC PR: #326
Ember Issue: (leave this empty)

---

# Ember Data Filter Deprecation

## Summary

Deprecate the `store.filter` API. This API was previously gated
behind a private `ENV` variable that was enabled by the addon
[`ember-data-filter`](https://github.com/ember-data/ember-data-filter/tree/b62c992186c00dce8cc81f1fb0cf5e2e6fee0f6b#ember-data-filter).

## Motivation

The `filter` API was a "memory leak by design". [Patterns exist](https://github.com/ember-data/ember-data-filter#recommended-refactor-guide)
with no-worse ergonomics that have better performance and do not incur memory leak penalties.
 
While the change in ergonomics for end consumers in minimal, the change to `ember-data` is substantial.
The code for this feature required significant amounts of confusing internal plumbing to ensure that
filters were rerun every time any form of mutation (update, addition, deletion) occurred to any record.

In addition to maintenance costs, this plumbing negatively affects the performance of all `RecordArray`s,
 and slow any operations that count as mutations (such as pushing new records into the store).

By removing this feature, we significantly simplify and streamline the core of `Ember Data`.

## Detailed design

We will provide 3 new deprecations with links to a [guide on how to refactor](https://github.com/ember-data/ember-data-filter#recommended-refactor-guide).
These deprecations will target `3.5`, meaning that the `ember-data-filter` addon will continue to
work and be supported through the release of ember-data `3.4`.

**Deprecation: ember-data-filter:filter**

Deprecate the primary case (`store.filter('posts', filterFn)`).
Instead, users can combine `store.peekAll` with a computed property.

**Deprecation: ember-data-filter:query-for-filter**

This deprecation is specific to folks providing a `query` to be requested the
first time a filter is run. To do this better, users can separate their usage
of `filter` from their usage of `query`.

**Deprecation: ember-data-filter:empty-filter**

In the case that users were creating a `filter` with no method for filtering by,
a deprecation is printed letting them know that the easiest path forward is to
use `peekAll`, which would return the same record result set.

## How we teach this

The `filter` API is rarely used, having been discouraged for many years. A simple post
 alerting users to it's deprecation should be sufficient. The refactoring guide is
 sufficiently simple that teaching folks a better way should not be much of a hurdle.

## Drawbacks

Minor churn for folks that did use this API; however, the end result will improve the
performance of apps using filters more so than anyone else.

## Alternatives

There's been some talk of an API for local querying; however, said alternative RFC
 would only result in deprecating this API as well.

## Unresolved questions

None
