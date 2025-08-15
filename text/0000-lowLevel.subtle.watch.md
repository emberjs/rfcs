---
stage: accepted
start-date: 2025-08-15T00:00:00.000Z
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

# RFC: lowLevel.subtle.watch

## Summary

Introduce a new low-level API, `lowLevel.subtle.watch`, available from `@ember/renderer`, which allows users to register a callback that runs when tracked data changes. This API is designed for advanced use cases and is not intended for general application reactivity.

It is not a replacement for computed properties, autotracking, or other high-level reactivity features.

> [!CAUTION]
> This is not a tool for general use. 

[tc39-signals]: https://github.com/tc39/proposal-signals

## Motivation

Some advanced scenarios require observing changes to tracked data without triggering a re-render or scheduling a revalidation. The `lowLevel.subtle.watch` API provides a mechanism for users to hook into tracked data changes at a low level, similar to [TC39's signals + watchers proposal][tc39-signals]:

Use cases include:
- synchronizing external state whithout the need to piggy-back off DOM-rendering
- ember-concurrency's `waitFor` witch would not need to rely on observers (as today) or polling
- Building alternate renderers (instead of rendering to DOM, render to `<canvas>`, or the Terminal)

> [!CAUTION]
> This API is not intended for application logic.

## Detailed design

### API Signature

```ts
type Unwatch = () => void;
function watch(callback: () => void): Unwatch;
```

The API is available as `lowLevel.subtle.watch` from `@ember/renderer`.

### Lifecycle and Semantics

- The callback runs during the transaction in `_renderRoot`, piggybacking on infrastructure that prevents sets to tracked data during render
  - This is not immediately after tracked data changes, nor during `scheduleRevalidate` (which is called whenever `dirtyTag` is called)
  - Callbacks are registered and run after tracked data changes, but before the next render completes
- Multiple callbacks may be registered; all will run in the same serially in relation to each other (if the same tracked data changes)
- Callbacks must not mutate tracked data. Attempting to do so will throw an error -- the backtracking re-render protection message.

### Safeguards

- If a callback attempts to set tracked data, an error is thrown to prevent feedback loops and maintain render integrity
- Callbacks are run in a controlled environment, leveraging Ember's transaction system to avoid side effects
- This API is intended for low-level integrations and debugging tools, not for general application logic

### Comparison to TC39 Signals/Watchers

- TC39's  `watch` proposal allows observing changes to signals in JavaScript
- Ember's `lowLevel.subtle.watch` is similar in spirit but scoped to tracked properties and the rendering lifecycle
- This API does not provide direct access to the changed value or path; it is a notification mechanism only
- Unlike TC39 watchers, this API is tied to Ember's render transaction system
  - if/when [TC39 Signals][tc39-signals] are implemented, the implementation of this behavior can be swapped out for the native implementation (as would be the case for all of Ember's reactivity)
- `watch` will return a method to `unwatch` which can be called at any time and will remove the callback from the in-memory list of callbacks to keep track of.

### Example

```gjs
import { lowLevel } from '@ember/renderer';
import { cell } from '@ember/reactive';

const count = cell(0);
const increment = () => count.current++;

lowLevel.subtle.watch(() => {
  // This callback runs when tracked data changes
  console.log('Tracked data changed! :: ', count.current);
  
  // Forbidden operations: setting tracked data
  // count.current = 'new value'; // Will throw!
});

<template>
  <button onclick={{increment}}>++</button>
</template>
```

## How we teach this

Until we prove that this isn't problematic, we should not provide documentation other than basic API docs on the export.

We do want to encourage intrepid developers to explore implementation of other renderers using this utility.

### Terminology

- "Subtle" indicates low-level, non-intrusive observation
- "Watch" aligns with TC39 Signals terminology and developer expectations
- "lowLevel" namespace clearly indicates advanced/internal usage

## Drawbacks

- Exposes internals that may be misused for application logic
- May increase complexity for debugging and maintenance
- Lack of unsubscribe mechanism may lead to memory leaks if misused
- May encourage patterns that bypass Ember's intended reactivity model
- Could be confused with higher-level reactivity APIs despite "subtle" naming

## Alternatives

n/a (for now / to start with)

## Unresolved questions

n/a