---
stage: accepted
start-date: 2025-08-13T14:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
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

<!-- Replace "RFC title" with the title of your RFC -->

# Add a `skipCopy` option to tracked-collections

## Summary

Add an option for all tracked-collections (like `trackedArray`, `trackedMap` and `trackedObject`) so that their constructors don't perform a copy of the passed-in collection.

## Motivation

That would allow users who know what they're doing to skip a potentially expensive copy of a big collection.

## Detailed design

The classes that implement the tracked-collections have an `options` argument. We can add an option to it to skip copying the collection argument that is passed.

The naming is of course the biggest issue. One might argue that it's good to have "unsafe" in the name because that really is unsafe - if the original collection is modified and it is not copied - that change will not be reflected in the template which might be confusing for the user. On the other hand, this is somewhat "obvious" so naming it "unsafe" might prevent people from using it and therefore skipping on potential (big) performance gains.

There is a [PR](https://github.com/glimmerjs/glimmer-vm/pull/1767) in the Glimmer VM repo for an implementation in `trackedArray`.

## How we teach this

We add it to the documentation. And also to the TypeScript types (which is already done in the PR above).

## Drawbacks

Potential confusion in people why their templates are not modified when the original collection is modified.

## Alternatives

I don't think there are. For the best performance, no copy should be made.

## Unresolved questions

None.
