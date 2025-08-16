---
stage: accepted
start-date: 2025-08-16T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

# Move tracked collections to `@ember/reactive/collections`

## Summary

This RFC proposes moving the tracked collections (`trackedArray`, `trackedSet`, `trackedMap`, `trackedWeakSet`, `trackedWeakMap`, and `trackedObject`) from the main `@ember/reactive` package to a subpath import at `@ember/reactive/collections`.

While it's true that dead-code-elimination would prevent shipping code users don't use, this organization may better communicate what is a core "reactive" thing, vs what is a helpful datastructure.

## Motivation

RFC 1068 introduced tracked collections into the `@ember/reactive` package, but placing them in the main export surface alongside core primitives like `tracked`, `cached`, `cell`, and `resource` may not conflate importance of the collections. 

> [!NOTE]
> At the time of writing of this RFC, `tracked`, `cached`, `cell`, and `resource` RFCs have not been accepted for inclusion in `@ember/reactive`

This is somewhat motivated by actual usage out in the ecosystem of tracked-built-ins:

here are results from github.com searches for the `tracked-built-ins` equivelents:
- "new TrackedObject(" - [888 Results](https://github.com/search?q=%22new+TrackedObject%28%22&type=code)
- "new TrackedArray(" - [468 Results](https://github.com/search?q=%22new+TrackedArray(%22&type=code)
- "new TrackedSet(" - [129 Results](https://github.com/search?q=%22new+TrackedSet(%22&type=code)
- "new TrackedWeakSet(" - [29 Results](https://github.com/search?q=%22new+TrackedWeakSet(%22&type=code)
- "new TrackedWeakMap(" - [32 Results](https://github.com/search?q=%22new+TrackedWeakMap(%22&type=code)

## Detailed design

Instead of importing tracked collections from the main `@ember/reactive` package:

```js
// Current (RFC 1068)
import { 
    trackedObject, trackedArray, 
    trackedMap, trackedWeakMap,
    trackedSet, trackedWeakSet
} from '@ember/reactive';
```

Users would import them from the `/collections` subpath:

```js
// Proposed
import { 
    trackedObject, trackedArray, 
    trackedMap, trackedWeakMap,
    trackedSet, trackedWeakSet
} from '@ember/reactive/collections';
```

This is non-breaking, because `@ember/reactive` hasn't been released yet.

## How we teach this

Same as in RFC #1068, but with updated import paths.

## Drawbacks

Developers need to remember that collections are imported from `@ember/reactive/collections` rather than the main `@ember/reactive` package, adding one more import path to the mental model.

This maintains the status quo tho as folks are already used to importing from `tracked-bulit-ins` for these collections.

## Alternatives

- Keep collections in main `@ember/reactive` module

## Unresolved questions
n/a