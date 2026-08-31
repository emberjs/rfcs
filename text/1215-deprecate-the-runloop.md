---
stage: accepted
start-date: 2026-08-04T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - learning
  - typescript
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

# Deprecate the runloop (`@ember/runloop`)

## Summary

The runloop (backburner) predates Promises, `async`/`await`, autotracking, and pretty much every scheduling primitive the platform has shipped since 2011.
Rendering no longer needs it -- autotracking schedules revalidation on its own -- so the only thing `@ember/runloop` does today is provide a worse, framework-specific way to do what the platform already does.

This RFC proposes deprecating all exports of `@ember/runloop`: `run`, `schedule`, `scheduleOnce`, `later`, `next`, `once`, `debounce`, `throttle`, `join`, `bind`, `begin`, `end`, `cancel`, and `getCurrentRunLoop`.

## Motivation

The runloop is one of the largest remaining sources of "you just have to know" knowledge in Ember:

- **It's redundant.** Since Octane, reactivity is autotracked. Setting a `@tracked` property schedules a render without anyone calling `schedule('render', ...)`. The queues (`sync`, `actions`, `routerTransitions`, `render`, `afterRender`, `destroy`) are an implementation detail of a world where the framework had to manually batch work.
- **It's a teaching burden.** "What is the runloop and when do I need `run()`?" has been a top interview-trivia / confusion generator for a decade. The [entire guides page](https://guides.emberjs.com/release/applications/run-loop/) on the runloop opens by telling you that you probably don't need to know about it.
- **It's a testing burden.** The "You have turned on testing mode, which disabled the run-loop's autorun" error is many developers' first (and worst) introduction to the runloop.
- **It has native replacements.** `setTimeout`, `requestAnimationFrame`, `queueMicrotask`, `Promise`, `async`/`await`, `AbortController` -- every use-case is covered by the platform, and settledness is covered by [`@ember/test-waiters`](https://github.com/emberjs/ember-test-waiters).
- **It costs bytes.** backburner.js and its integration ship to every user of every Ember app, regardless of whether the app uses any of it.

This is a part of _[Deprecating Ember Classic (pre-Octane)](https://github.com/emberjs/rfcs/issues/832)_.

## Relationship to RFC #957 (Render Aware Scheduler Interface)

[RFC #957](https://github.com/emberjs/rfcs/pull/957) shares this RFC's entire diagnosis: backburner predates microtasks, `requestAnimationFrame`, and native promises; polyfilled task queues ruin stack traces and profiler output; and interleaving RSVP/backburner flushes with native async causes extraneous renders, DOM thrash, and backtracking-render hazards. On the "why", the two RFCs are the same RFC.

They differ on the "what instead":

- **RFC #957** replaces `@ember/runloop` with a new public scheduling *interface* -- work is scheduled by declaring intent relative to the browser's frame lifecycle (`tasks`, `render`, `layout`, `composite`, `next`, `idle`), with pluggable `Strategy` implementations.
- **This RFC** replaces `@ember/runloop` with *nothing*. Platform primitives, modifiers, destroyables, and test waiters cover the use-cases that actually appear in apps and addons, and Ember's own render scheduling stays internal and non-public.

These are not in conflict -- deprecating `@ember/runloop`'s public API is the shared first step of both proposals, and this RFC deliberately takes no position that would block #957. If a coordinated read/write phase model proves necessary (the `layout` / `composite` phases are the part the platform genuinely does not give you, for avoiding layout thrash during animation-heavy work), that interface can ship later -- as a standalone package or a follow-up RFC -- and it would be *easier* to ship into an ecosystem that has already migrated off runloop semantics, because the new scheduler wouldn't need to interoperate with backburner's queues, timers, or autorun behavior.

Put differently: #957 is "replace the runloop with a better scheduler"; this RFC is "remove the runloop, then find out whether we still need a scheduler at all." The migration work asked of app and addon authors is nearly identical in both, so no effort spent following this RFC's transition path is wasted if #957 (or a successor) is accepted later.

## Transition Path

Ecosystem usage of each API, for gauging impact:

|   | API | Usage: EmberObserver |
| - | --- | ----- |
|🌐 | `run` | [Many, mostly in tests](https://emberobserver.com/code-search?codeQuery=%20run(&sort=updated&sortAscending=false) |
|🌐 | `later` | [Many](https://emberobserver.com/code-search?codeQuery=runloop%27%3B&sort=updated&sortAscending=false) |
|🌐 | `next` | [Many](https://emberobserver.com/code-search?codeQuery=next(&sort=updated&sortAscending=false) |
|🌐 | `schedule` | [Many, mostly `afterRender`](https://emberobserver.com/code-search?codeQuery=schedule(%27afterRender&sort=updated&sortAscending=false) |
|🌐 | `scheduleOnce` | [Many](https://emberobserver.com/code-search?codeQuery=scheduleOnce(&sort=updated&sortAscending=false) |
|🌐 | `once` | [Some](https://emberobserver.com/code-search?codeQuery=once(&sort=updated&sortAscending=false) |
|🌐 | `debounce` | [Many](https://emberobserver.com/code-search?codeQuery=debounce(&sort=updated&sortAscending=false) |
|🌐 | `throttle` | [Some](https://emberobserver.com/code-search?codeQuery=throttle(&sort=updated&sortAscending=false) |
|🌐 | `join` | [Some, mostly older addons](https://emberobserver.com/code-search?codeQuery=join(&sort=updated&sortAscending=false) |
|🌐 | `bind` | [Few](https://emberobserver.com/code-search?codeQuery=bind(&sort=updated&sortAscending=false) |
|🌐 | `begin` / `end` | [Few, very old](https://emberobserver.com/code-search?codeQuery=begin()&sort=updated&sortAscending=false) |
|🌐 | `cancel` | [Many (paired with `later`/`debounce`)](https://emberobserver.com/code-search?codeQuery=cancel(&sort=updated&sortAscending=false) |
|🫣 | `getCurrentRunLoop` | [Few, test-support code](https://emberobserver.com/code-search?codeQuery=getCurrentRunLoop&sort=updated&sortAscending=false) |

The overall shape of the migration:

1. anything that _schedules work_ moves to a platform primitive (`setTimeout`, `requestAnimationFrame`, `queueMicrotask`, `await`),
2. anything that needs _settledness in tests_ registers a [test waiter](https://github.com/emberjs/ember-test-waiters),
3. anything that needs _cleanup_ uses [`@ember/destroyable`](https://api.emberjs.com/ember/release/modules/@ember%2Fdestroyable) (or a library that composes the two, like [ember-lifeline](https://github.com/ember-lifeline/ember-lifeline) or [ember-concurrency](https://ember-concurrency.com)).

For (2), we should provide a `runloop-to-platform` codemod for the mechanical cases (`later` → `setTimeout`, `next` → `setTimeout(fn, 0)`, `bind` → arrow fn), but most of the value here is deleting code, not rewriting it.

### Scenario: `run` wrapping (mostly in tests)

Before:
```js
import { run } from '@ember/runloop';

run(() => {
  this.model.set('name', 'NVP');
});
```

After:
```js
this.model.set('name', 'NVP');
```

That's it. With autotracking, there is no batching to opt in to -- rendering is scheduled automatically, and `await settled()` (or any `@ember/test-helpers` interaction helper) flushes it in tests.

The "autorun assertion" that made `run()`-wrapping necessary in tests will be removed as part of implementing this RFC, as it exists to protect a constraint (all work must happen inside a runloop) that no longer exists.

### Scenario: `later` / `cancel`

Before:
```js
import { later, cancel } from '@ember/runloop';

class Poller extends Service {
  start() {
    this.timer = later(this, this.poll, 1000);
  }

  willDestroy() {
    cancel(this.timer);
  }
}
```

After:
```js
import { registerDestructor } from '@ember/destroyable';

class Poller extends Service {
  start() {
    let timer = setTimeout(() => this.poll(), 1000);

    registerDestructor(this, () => clearTimeout(timer));
  }
}
```

or, with ember-concurrency (which also handles the test-waiter and destruction concerns for you):

```js
import { task, timeout } from 'ember-concurrency';

class Poller extends Service {
  poll = task(async () => {
    await timeout(1000);
    // ...
  });
}
```

### Scenario: `next`

`next` was almost always used to mean "after the thing that is currently happening finishes."

Before:
```js
import { next } from '@ember/runloop';

next(() => this.focusInput());
```

After (microtask -- sufficient for "after this synchronous work"):
```js
await Promise.resolve();
this.focusInput();
```

After (macrotask -- what `next` actually did):
```js
setTimeout(() => this.focusInput(), 0);
```

### Scenario: `schedule('afterRender')`

Before:
```js
import { schedule } from '@ember/runloop';

schedule('afterRender', this, this.measureElement);
```

After -- in almost every case, this code is trying to do something with a DOM node after it exists, which is exactly what a [modifier](https://api.emberjs.com/ember/release/modules/@ember%2Fmodifier) is for:

```gjs
import { modifier } from 'ember-modifier';

const measure = modifier((element) => {
  // element is rendered and in the DOM
});

<template>
  <div {{measure}}>...</div>
</template>
```

If you truly need "after the browser paints", that's `requestAnimationFrame` (and was never actually guaranteed by `afterRender` anyway).

Scheduling into any other queue (`sync`, `actions`, `routerTransitions`, `render`, `destroy`) is scheduling against framework internals, and has no supported equivalent -- on purpose.

### Scenario: `debounce` / `throttle`

Before:
```js
import { debounce } from '@ember/runloop';

@action
onScroll() {
  debounce(this, this.updatePosition, 100);
}
```

After, with ember-concurrency:
```js
import { task, timeout } from 'ember-concurrency';

updatePosition = task({ restartable: true }, async () => {
  await timeout(100);
  // ...
});
```

or any standalone implementation ([`lodash.debounce`](https://lodash.com/docs#debounce), [`p-debounce`](https://github.com/sindresorhus/p-debounce), or ~10 lines of your own around `setTimeout`). The runloop versions were never better than these; they were just already in your bundle.

### Scenario: `bind`

Before:
```js
import { bind } from '@ember/runloop';

element.addEventListener('scroll', bind(this, this.onScroll));
```

After:
```js
element.addEventListener('scroll', () => this.onScroll());
```

`bind` existed to re-enter the runloop from "outside" (native event handlers, `postMessage`, websockets, jQuery plugins). With no runloop to re-enter, an arrow function (or `@action` for `this`-binding) is all that's left to do.

### Scenario: `join`, `begin` / `end`, `getCurrentRunLoop`

These exist only to interoperate with the runloop itself (join an in-flight loop, manually open/close one, introspect one). Once nothing requires being "inside a runloop", there is nothing to join, begin, end, or get.

Any code doing:
```js
if (getCurrentRunLoop()) {
  join(() => this.flush());
} else {
  run(() => this.flush());
}
```

becomes:
```js
this.flush();
```

### Settledness: the actual hard part

The one real service the runloop provided was implicit test settledness: `await settled()` knows about pending `later` timers because backburner tracks them.
`setTimeout` is invisible to `settled()`.

The replacement is explicit, and better -- [`@ember/test-waiters`](https://github.com/emberjs/ember-test-waiters):

```js
import { buildWaiter } from '@ember/test-waiters';

const waiter = buildWaiter('my-app:poller');

function pollLater(fn, ms) {
  let token = waiter.beginAsync();

  return setTimeout(() => {
    waiter.endAsync(token);
    fn();
  }, ms);
}
```

"Explicit" is a feature here: backburner's implicit tracking is also why a stray `later(this, this.poll, 30_000)` makes your test suite hang for 30 seconds. With test waiters, *you* decide what settledness means for your async. Libraries like ember-concurrency and ember-lifeline already integrate with test waiters, so using them gets you this for free.

### Implementation Plan

1. Remove the internal coupling: ensure nothing in `ember-source` requires user code to run inside a runloop (rendering already doesn't; the remaining bits are event dispatch, router transition batching, and the destroy queue -- these move to microtask/autotracked scheduling internally).
2. Remove the autorun assertion in testing mode.
3. Add deprecations (`id: deprecate-runloop`, one sub-id per export, `until: 8.0.0`) to each `@ember/runloop` export.
4. Publish the deprecation guide entries (largely the _Transition Path_ scenarios above).
5. Provide the codemod for the mechanical transforms.
6. Add an `eslint-plugin-ember` rule flagging imports from `@ember/runloop` (on by default in the recommended config once the deprecation ships).

`ember-source` will keep backburner as a (non-public) internal detail for as long as its own internals need it; this RFC deprecates the *public API* and the *programming model*, which is what blocks apps and addons.

## How We Teach This

Mostly, we get to *un*teach:

- Delete the [run loop guides page](https://guides.emberjs.com/release/applications/run-loop/) and the [backburner debugging section](https://guides.emberjs.com/release/configuring-ember/debugging/#toc_errors-within-emberrunlater-backburner).
- Remove `@ember/runloop` from the API docs (after the deprecation lands, with a banner pointing at the deprecation guide in the interim).
- The guides' async examples already use `async`/`await`; the testing guides already use `await settled()` and interaction helpers.

The new content needed is a good deprecation guide (the scenarios above) and a guides section on `@ember/test-waiters`, which is currently under-documented relative to how important it is.

## Drawbacks

- As with any deprecation, we introduce an upgrade cliff for addons that are updated infrequently, and consequently their consuming apps. The runloop is old and load-bearing, so the tail here is long.
- Settledness regressions are quiet: code migrated from `later` to a bare `setTimeout` without a test waiter doesn't fail loudly -- tests get flaky instead. The lint rule and deprecation guide need to hammer on this.
- `debounce`/`throttle` migrations require picking a userland implementation, which is a (small) decision where there used to be a default.

## Alternatives

- Do nothing. The cost of keeping the runloop is:
  - every Ember developer eventually pays the "what is the runloop" tax
  - backburner ships in every app forever
  - test-mode autorun errors keep being the first impression of Ember's testing story
  - internals keep maintaining ordering guarantees (queue semantics) that nothing modern relies on
- Deprecate only the obviously-dead exports (`begin`, `end`, `join`, `bind`, `getCurrentRunLoop`) and keep the timer conveniences (`later`, `debounce`, `throttle`, `cancel`). This keeps backburner in every bundle to provide utilities that npm and the platform provide better, and keeps the implicit-settledness trap around.
- Adopt [RFC #957](https://github.com/emberjs/rfcs/pull/957)'s scheduler interface as the designated replacement, and make this deprecation part of that migration. This is a viable path (see _[Relationship to RFC #957](#relationship-to-rfc-957-render-aware-scheduler-interface)_), but it couples "stop shipping backburner" to "design, implement, and stabilize a new public scheduler" -- the deprecation shouldn't have to wait on the harder of the two projects, and most runloop usage migrates to the platform, not to a scheduler.
- Move the timer utilities to a standalone `@ember/timers` package with test-waiter integration built in. This is really just "bless ember-lifeline", which already exists -- the ecosystem doesn't need `ember-source` to own it.

## Unresolved questions

- Exact ordering with the interop work in step 1 of the implementation plan -- some internal consumers (legacy `@ember/component` event dispatch, in particular) may force the runloop deprecation to sequence after other Ember Classic deprecations land.
- Should `ember-source` provide a first-party `settled()`-aware `waitFor`-style timer helper in `@ember/test-waiters`, so the `later` → `setTimeout` migration has a zero-thought default? (Leaning yes.)
