---
stage: accepted
start-date: 2023-09-09T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/957
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

# Render Aware Scheduler Interface

## Summary

This RFC Proposes replacing `@ember/runloop` [Backburner.js](https://github.com/BackburnerJS/backburner.js)
with an [interface](https://www.typescriptlang.org/docs/handbook/interfaces.html) for common scheduling needs.

The interface describes *intent* for when work should be performed in relation to the native
event queues and render cycle of the browser. The details of *how* that work is scheduled and
flushed are up to the specific implementation, allowing for experiments in this space.

Additionally, this RFC proposes deprecations and alterations to associated async primitives
such as RSVP to support this exploration.

## Motivation

### The Short Answer: *Better Things Now Exist*

Backburner (`@ember/runloop`) was written in an era that predates most async primitives including [microtasks](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API/Microtask_guide) and [requestAnimationFrame](https://developer.mozilla.org/en-US/docs/Web/API/window/requestAnimationFrame).

It was written to help coordinate work that needed to complete before the browser might render again: in effect this made backburner a microtask polyfill, albeit one with requestAnimationFrame ideals.

[RSVP](https://github.com/tildeio/rsvp.js) similarly was written in an era in which native Promise support was being proposed but did not yet exist. As
such, while it provided an attempt at polyfilling microtasks, Ember chose to replace RSVP's microtask polyfill with
Backburner in order to reduce the overhead of promise flushing and reduce the risk of multiple renders occurring
within the same frame. In effect, this allowed Backburner to *also* function to [multiplex](https://en.wikipedia.org/wiki/Multiplexing)
Promises *and* to function as an *after microtasks complete* callback from which Ember's render work would be
flushed.

Noteably, the idea of being able to control the last microtask in the queue for this sort of framework level
work has become [a common need](https://twitter.com/jarredsumner/status/1694351991626166658?s=20), but remains
a missing primitive.

In short, to summarize the motivations of the *current* system, Ember had need of

- a high priority task queue
- a guarantee of the ability to do its render work before browser render
- the ability to let async work complete before flushing render
- a signal that work was occurring after which it should render
- the ability to let other work be scheduled to be done relative
  to its render.

### The Other Short Answer: *Embrace The Platform*

Polyfilled microtasks typically do not have a meaningful stack trace due to their queued flush,
resulting in debugging and error tracing being especially difficult to use.

Native microtasks automatically piece together the stack trace across async boundaries, allowing
in most situations an error stack or a pause in a debugger to allow the developer to work back to
an originating change more quickly. This includes in performance
tooling such as the chrome profiler, where promises and other async callbacks will draw a line back to the point they were scheduled to help explore how the work evolved.

Thus encouraging use of native microtasks whereever possible automatically improves the debugging
experience for all developers, espcially for remote traces captured by bugtracking software.

It also significantly simplifies the mental model and clarity for developers. While the concept of scheduling
will still exist, this RFC would enable developers to remove
"what is RSVP", "what is Backburner" and "what is the Runloop" from their mental model. Instead of thinking about scheduling, developers in most cases can "just use a promise" or "just use async/await" or "just use concurrency".

### The Complex Answer: *Making Mutation Safer & Performance*

As native promises, async/await usage, and raw fetch usage have become ever more prevalent,
the benefits of a "unified" flush provided by Backburner and the Backburner/RSVP configuration
have both diminished and even become largely detrimental.

Because Ember will re-render every time a flush occurs, interleaving native and RSVP promises
results in repeated re-renders while resolving a promise chain that ought to have produced a
single render. Similarly, every XHR/Fetch result produces its own render stack where before
requests that resolved within a very short time of each other would have usually had their
work scheduled into the same deferred backburner queue flush.

This results in not only a substantial amount of unnecessary and typically duplicate work
being done before the next meaningful render, but also results in thrashing the DOM and forced
layouts. In addition to poor performance, this becomes a disaster for accessibility tooling as
focus and DOM shifts rapidly around waiting for a final settled state.

Additionally, this interleaving means that there are three significant risks to application code
where async data fetching is involved:

1) Being notified due to an extraneous render to calculate too early before the data is in the
right settled state. This can result in errors due to missing or incomplete state that should
have been able to have been depended on to be there by the time of render.

2) Mutating state multiple times due to extraneous renders, leading to higher potential for
hitting the dreaded "backtracking render" error when it should have been safe to have read and
written multiple times.

3) Awaiting a data fetch and assuming that the work done after await occurs before render, when
due to Ember's current zealous flushing it is taking place post-render.


## Detailed design

The scheduler interface defines several "phases" of when work should be done
that align to concepts in the browser's render cycle. The primary goal is
to reduce the number of times Ember needs to alter the DOM per frame to 0 or 1
as often as possible.

We also want to help applications coordinate work in a way that avoids the pitfalls
of DOM read/write interleaving. This means we need to provide both a mechanism for
applications to schedule work that occurs *after* the render but before the browser
has painted, and a mechanism by which to schedule work *after* that work but also
before the browser has painted.

To understand the phases of work to be done, it helps to understand a bit about
the mechanics of the browser's render cycle.

### Frames

The scheduler interface conceptualizes work as belonging to a "Frame", where a Frame
constitutes the time between when states of the DOM are observable to a user.

Below is a rough approximation of how browsers schedule various work.

![Frame Overview](../images/0957-frame-overview.svg)

Tasks are things such as `setTimout`, callbacks for `DOMEvent`s, or the completion
of an `xhr` or `fetch` request. The browser will execute as many tasks as it can
until it has met or passed the point at which it would like to try to update the screen.

`Microtasks` are callbacks executed via  `createMicrotask(callback)`, promise completion
such as `Promise.resolve().then(callback)`, `MutationObserver` callbacks, or `MessageChannel`
callbacks. Microtasks can recursively schedule new microtasks, and the browser will not
move on to another task or continue towards updating the screen until the queue is exhausted.

`FrameTasks` are callbacks executed for `requestAnimationFrame(callback)`. The browser
will execute all `FrameTasks` that existed when it began flushing the queue, but callbacks
may not schedule new `FrameTasks` recursively. E.g. any new calls to `requestAnimationFrame`
will go into the "next" Frame.

Similar to `FrameTasks`, `ResizeTasks` are for `ResizeObserver` callbacks and `IntersectionTasks`
are for `IntersectionObserver` callbacks. These queues can recursively schedule, though if
too much recursion is encountered by a `ResizeObserver` is errors and tries again on the next frame.

### Phases

With this understanding of the lifecycle of a Frame, roughly speaking we start to see a few
patterns for how we might want to schedule work in a way that seamlessly coordinates with the
framework and other parts of our application.

We divide this work into 6 conceptual phases:

- `tasks` - work that updates reactive state
- `render` - work that may need to adjust reactive state that needs to occur after Ember has updated the DOM and run any associated modifiers but before the browser shows the new state
- `layout` - work that needs to read DOM but does not require adjusting reactive state and should occur after `render` but before the browser shows the new state.
- `composite` - work that needs to write DOM but does not require reading state or adjusting reactive state and should occur after `render` and `layout` but before the browser shows the new state. E.g. adjusting transform values during an animation.
- `next` - work that should be deferred as a `task` to be done only once the browser has completed the current frame.
- `idle` - work that should be deferred until the browser is under less load.

We will discuss these phases in more depth below.

### Strategies

We refer to an implementation of the scheduler interface as a `Strategy`. The
strategy gets to shoose when each promise will resolve, and what happens if
say `render` is invoked while `layout` is flushing.

```ts
interface Strategy {
  render(): Promise<void>;
  layout(): Promise<void>;
  composite(): Promise<void>;
  next(): Promise<void>;
  idle(): Promise<void>;
}
```

Notably the strategy has no knowledge of the work to be done. This keeps the scheduling
overhead light, and enables async stack traces for scheduled work to maintain the context
of where the work was scheduled.

For instance, take the partial strategy implementation shown below which handles `render`
`layout` and `composite` by scheduling three `requestAnimationFrame` callbacks wrapped in
promises and allows recursive calls to render and just-in-time calls to layout and composite.

**Example 1**
```ts
class Scheduler {
  _nextRender = null;
  _nextLayout = null;
  _nextComposite = null;
  _isFlushing = false;
  _isFlushingRender = false;
  _flushComplete = null;

  ensureFrame(name) {
    if (!this._nextRender) {
      if (this._isFlushingRender) {
          this._nextRender = Promise.resolve();
      } else if (!this._isFlushing || name === 'render') {
        this._nextRender = new Promise(resolve => {
          requestAnimationFrame(() => {
            console.log('flushing render');
            this._isFlushing = true;
            this._isFlushingRender = true;
            this._nextRender = null;
            resolve();
          });
        });
      }
    }

    if (!this._nextLayout) {
      if (!this._isFlushing || name === 'layout') {
        this._nextLayout = new Promise(resolve => {
          requestAnimationFrame(() => {
            console.log('flushing layout');
            this._isFlushingRender = false;
            this._nextRender = null;
            this._nextLayout = null;
            resolve();
          });
        });
      }
    }

    if (!this._nextComposite) {
      this._nextComposite = new Promise(resolve => {
        requestAnimationFrame(() => {
          console.log('flushing composite');
          this._nextComposite = null;
          resolve();
        });
      });
    }

    if (!this._flushComplete) {
      this._flushComplete = new Promise(resolve => {
        requestAnimationFrame(() => {
          console.log('flushing complete');
          this._isFlushingRender = false;
          this._isFlushing = false;
          this._flushComplete = null;
          resolve();
        });
      });
    }
  }

  render() {
      this.ensureFrame('render');
      return this._nextRender;
  }

  layout() {
      this.ensureFrame('layout');
      return this._nextLayout;
  }

  composite() {
      this.ensureFrame('composite');
      return this._nextComposite;
  }
}
const scheduler = new Scheduler();
```

Using this strategy, lets observe what happens when we use normal `async/await`
patterns to schedule some work.

**Example 2**
```ts

// puts some blocking named work into the performance profiler
// to make it more obvious where the work was done and what work it was
function doExpensiveWork(name, step) {
  const fn = new Function(`return function ${name}() {const start = performance.now(); while (performance.now() - start < 10) {} console.log('${step}. ${name}');};`);
  return fn();
}      

async function doWork() {
  requestAnimationFrame(() => { doExpensiveWork('before', 1)() });
  let render = scheduler.render();
  requestAnimationFrame(() => { doExpensiveWork('after', 7)() });
  await render;
  doExpensiveWork('render', 2)();
  await Promise.resolve();
  doExpensiveWork('promise', 3)();
  await scheduler.render();
  doExpensiveWork('renderAgain', 4)();
  await scheduler.layout();
  bdoExpensiveWork('layout', 5)();
  await scheduler.composite();
  doExpensiveWork('composite', 6)();
}

doWork();

// Output:
// 1. before
// flushing render
// 2. render
// 3. promise
// 4. render again
// flushing layout
// 5. layout
// flushing composite
// 6. composite
// 7. after
```

And a more complicated example with interleaving:

**Example 3**
```ts
requestAnimationFrame(() => { doExpensiveWork('before', 0)() });
scheduler.render()
  .then(() => {
    doExpensiveWork('render', 1)();

    scheduler.composite()
      .then(() => {
        doExpensiveWork('composite', 5)();
      })
  });
requestAnimationFrame(() => { doExpensiveWork('after', 8)() });

scheduler.render()
  .then(() => {
    doExpensiveWork('render', 2)();

    scheduler.composite()
      .then(() => {
        doExpensiveWork('composite', 6)();
      });

    return scheduler.composite();
  })
  .then(() => {
    doExpensiveWork('composite', 7)();
  });

scheduler.composite()
  .then(() => {
    doExpensiveWork('composite', 4)();
  });

scheduler.layout()
  .then(() => {
    doExpensiveWork('layout', 3)();
    scheduler.layout()
      .then(() => { doExpensiveWork('layout', 9)(); });
  });

// Output:
// 0. before
// flushing render
// 1. render
// 2. render
// flushing layout
// 3. layout
// flushing composite
// 4. composite
// 5. composite
// 6. composite
// 7. composite
// flushing complete
// 8. after
// 9. layout
```

### The Default Strategy

```ts
import strategy from '@ember/scheduler/strategy';
```
### Providing a Strategy

```ts
import { registerStrategy } from '@ember/scheduler';
```

The scheduling strategy should be provided by registering it when defining the Application.

```ts
import Application from '@ember/application';
import { registerStrategy } from '@ember/scheduler';
import Resolver from 'ember-resolver';
import loadInitializers from 'ember-load-initializers';
import config from 'test-embroider/config/environment';

// the default scheduler implementation
import strategy from '@ember/scheduler/strategy';

export default class App extends Application {
  modulePrefix = config.modulePrefix;
  podModulePrefix = config.podModulePrefix;
  Resolver = Resolver;
}

registerStrategy(strategy);
loadInitializers(App, config.modulePrefix);
```

### Scheduling Work Into a Phase

Each phase has a corresponding import fom `@ember/scheduler` for 
use in scheduling in the app.

```ts
import { render, layout, composite, next, idle } from '@ember/scheduler';
```

Each import is a function returning a promise that resolves according
to the strategy of the configured strategy.

```ts
await render();
```

This allows scheduling from any function, class or context irregardless
of ability to access an ember service.

### Cancelling Work

Since the scheduler does not itself store any callbacks, there is no need
to tell the scheduler to cancel work. Instead, if your work requires cancellation
or cleanup, handle this at the point the work was scheduled.

```ts
class extends Component {
  @tracked width= 10;

  doWork() {
    this.width = 100;
    await render();
    if (this.isDestroyed) {
      return;
    }
  }
}
```

Libraries or the framework may desire to provide sugar for automated cleanup,
and can do so over this much simpler primitive.

### The Phases In Detail

#### Phase 1: Tasks

In this first phase, state is read and modified as a reaction to either
a user interaction, network request completion, or timer callback.

These "tasks" and their associated microtasks "simply just work". Users
need to do no special scheduling (such as the runloop's former `schedule('actions', callback)`
and `schedule('routerTransitions', callback)` queues), and need to use zero framework =
or library provided primitives (such as RSVP, or the runloop's `bind` `run` or `join` methods).

In order for this to work, Ember's reactivity system will be evolved to ensure that it
becomes responsible for scheduling an update to render whenever reactive state has changed.

This more or less means that the reactivity primitives we use become responsible for
triggering the render schedule or a revalidation, instead of using backburner's
`on('begin', callback)` to schedule a render any time a runloop is initiated as is the
case today, and `on('end', callback)` to schedule a revalidate (as it does today).

During the tasks phase, user code cannot expect the DOM to have updated to reflect changes to
state. This is similar to how the timing expectations work in the runloop today with `render`
and `afterRender`, but extends this expectation across potentially multiple tasks and
microtasks.

### Phase 2: Render

The render phase occurs whenever Ember has decided to render new DOM containing
the changes you've just made. Scheduling into `render` guarantees that your work has
access to that DOM prior to the next paint.

```ts
import { render } from '@ember/scheduler';

// ...

await render();
```

A Scheduler implementation may choose to flush render at any time, potentially
multiple times so long as the work completes prior to the next paint.

The magic here is in `@ember/scheduler`. It wraps the promise returned by the strategy
for `render`, allowing it to flush generation of DOM and execution of modifiers before
any `await render()` calls flush no matter when that might be.

During the render phase, updates to reactive state are allowed, but Ember does not
guarantee that any updates will rerender before the next paint, this is up to the
scheduler implementation to decide.

Writing DOM during this phase will error in development.

Both `schedule('render')` and `schedule('afterRender')` will be mapped to this phase
in the transition.

### Phase 3: Layout

The layout phase occurs after the render phase and prior to the next paint.

```ts
import { layout } from '@ember/scheduler';

// ...

await layout();
```

This phase is for work that needs to read DOM but does not require adjusting reactive state.

Writing DOM during this phase will error in development.
Ember should Error in this phase if reactive state is written.

### Phase 4: Composite

The composite phase occurs after the layout phase and prior to the next paint.

```ts
import { composite } from '@ember/scheduler';

// ...

await composite();
```

This phase is for work that needs to write DOM but does not require reading DOM state
or adjusting reactive state.

Users should take every opportunity to avoid reading DOM in this phase to avoid forced
layouts and interleaved read/write of DOM state.

This phase is ideal for updating animations or moving tooltips to a final position based
on measurements made in the previous phase.

Ember should Error in this phase if reactive state is written.

### Phase 5: Next

This phase is for work that needs to escape the current frame but is still a relatively
high priority.

```ts
import { next } from '@ember/scheduler';

// ...

await next();
```

### Phase 6: Idle

This phase is for work that is low priority. Most commonly tasks like background fetch, server pings, or analytics processing.

```ts
import { idle } from '@ember/scheduler';

// ...

await idle();
```

### Outline of Work To Be Done

- Remove configuration logic for RSVP from Ember, allow it to use its own polyfill without the runloop
- Deprecate RSVP in favor of a smaller util library for just things like `hash`
- Move "timers" (`throttle`, `debounce` and `later`) out of `@ember/runloop` into their own standalone
  library.
- Deprecate (no replacement) `run` `join` and `bind` from `@ember/runloop`
- Replace `schedule` and `next` with a new Scheduler interface
- Provide a default implementation of the new scheduler interface
- Remove `@types/ember__runloop`, `@types/rsvp`, `rsvp`, and `ember-fetch` from anywhere they still live in the default blueprint
  or as a dependency in a core package

### Implementation Strategy

- RSVP configuration would be gated behind an optional feature flag `use-native-rsvp-flush`, with a deprecation
    to set it to the new behavior
- RSVP usage would be deprecated at the import level at build time, using infra similar to ember-cli-babel deprecation
- ember-concurrency should remove its usage of the runloop and RSVP
- ember-concurrency should add a test-waiter to tasks
- Timer move would be handled via a normal deprecation + codemod to shift imports to the new import location
- an optional feature flag `use-async-scheduler` would move `scheduleOnce`, `schedule`, `next`, `run`, `join` and `bind` to delegating
  into the new scheduler interface.
  - `run`, `join` and `bind` would execute the callback given to them and do no more.
  - `next` and the named queues except for `actions` for `schedule` and `scheduleOnce` would be mapped onto the new scheduler
     interface and executed accordingly.
  - work scheduled into `actions` would run as a microtask right away: `schedule('action', doWork)` becomes `Promise.resolve().then(doWork)`
  - Ember's glimmer integration would shift from assuming that scheduling render in the render phase of every backburner flush and additionally validating that render at the end of every flush is "sync" to being aware that it is async and utilizing the new scheduler interface to schedule its render and its revalidate at the appropriate times. **This is one of the biggest reasons this migration is handled by an app-wide flag** as it carries the potential for apps to encounter bugs due to having become accidentally reliant on existing "sync complete" timing semantics.
- a deprecation would be printed for usage of any `@ember/runloop` API
- the ability to use zero-production-cost promise-based test-waiters quickly throughought code should be improved

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

Is `@ember/scheduler` the right name? Or is there utility in a non-Ember associated name?
