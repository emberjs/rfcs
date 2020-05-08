- Start Date: 2020-04-30
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/626
- Tracking: (leave this empty)

# JavaScript Helper Invocation API

## Summary

This RFC proposes a new API, `invokeHelper`, which can be used to invoke a
helper definition, creating an instance of the helper in JavaScript.

```js
// app/components/data-loader.js
import Component from '@glimmer/component';
import Helper from '@ember/component/helper';
import { invokeHelper } from '@ember/helper';

class FetchTask {
  @tracked isLoading = true;
  @tracked isError = false;
  @tracked result = null;

  constructor(url) {
    this.run(url);
  }

  async run(url) {
    try {
      let response = await fetch(url);
      this.result = await response.json();
    } catch {
      this.isError = true;
    } finally {
      this.isLoading = false;
    }
  }
}

class RemoteData extends Helper {
  compute([url]) {
    return new FetchTask(url);
  }
}

export default class DataLoader extends Component {
  data = invokeHelper(this, RemoteData, () => {
    positional: [this.args.url]
  });
}
```
```hbs
<!-- app/components/data-loader.hbs -->
{{#if this.data.value.isLoading}}
  Loading...
{{else if this.data.value.isError}}
  Something went wrong!
{{else}}
  {{this.data.value.result}}
{{/if}}
```

## Motivation

As Ember has evolved, the framework has been developing a model of reactive,
incremental computation. This model is oriented around _templates_, which map
data and JavaScript business logic into HTML that is displayed to the user.

On the first render, this mapping is fairly similar to standard programming
languages. You can imagine that every component is like a _function call_,
receiving arguments and data, processing it, and placing it within its own
template, ultimately producing HTML as the "return value" of the component.
Components can use other components, much like functions can call other
functions, resulting in a tree structure of nested components, which is
the application. At the root of our application is a single "main" component
(similar to the main function in many programming languages) which takes
arguments and returns the full HTML of the initial render.

Where Ember's programming model really begins to differ is in _subsequent_
renders. Rather than re-running the entire program whenever something changes,
producing new HTML as a result, Ember incrementally re-runs the portions of the
program which have changed. It knows about these portions via its change
tracking mechanism, _autotracking_.

This means fundamentally that the tree of components differs from a tree of
functions because components can _live longer_. They exist until the portion of
the program that they belong to has been removed by an incremental update, and
as such, they have a _lifecycle_. Unlike a function, a component can _update_
over time, and will be _destroyed_ at some unknown point in the future.

Components previously exposed this lifecycle directly via a number of
lifeycle hooks, but there were many issues with these hooks. These stemmed from
the fact that components were the smallest _atom_ for reactive composition. In
the world of functions, a piece of code can always be broken out into a new
function, giving the user the ability to extract repeated functionality,
abstracting common patterns and concepts and reducing brittleness.

```js
// before
function myProgram(data = []) {
  for (let item of data) {
    // ...
  }

  let values = data.map(() => {
    // ...
  });

  while (values.length) {
    let value = values.pop();
    // ...
  }
}
```
```js
// after
function initialProcessing(data) {
  for (let item of data) {
    // ...
  }
}

function extractValues(data) {
  return data.map(() => {
    // ...
  });
}

function processValues(values) {
  while (values.length) {
    let value = values.pop();
    // ...
  }
}

function myProgram(data = []) {
  initialProcessing(data);

  let values = extractValues(data);

  processValues(values);
}
```

In the reactive model of components, there often is not a way to do this
transparently, since the only portions of the code that are reactive are the
component hooks themselves. This results in related code being spread across
multiple locations, with the user being forced to keep the relationships between
these bits of code in their head at all times, and understand the interactions
between them.

```js
import Component from '@ember/component';
import { fetchData } from 'data-fetch-library';

export default class Search extends Component {
  // Args
  text = '';
  pollPeriod = 1000;

  didReceiveAttrs() {
    if (this._previousText !== this.text) {
      this._previousText = this.text;
      this.data = fetchData(`www.example.com/search?query=${this.text}`);
    }

    if (this._pollPeriod !== this.pollPeriod) {
      this._pollPeriod = this.pollPeriod;

      cancelInterval(this._intervalId);
      this._intervalId = setInterval(() => {
        this.data = fetchData(`www.example.com/search?query=${this.text}`);
      }, pollPeriod);
    }
  }

  willDestroy() {
    cancelInterval(this._intervalId);
  }
}
```

There are a few other constructs in templates that have the same reactive
lifecycle as components - helpers and modifiers. And helpers in particular are
very useful, because they can receive arguments like components, but they can
return _any_ value, not just HTML. The only issue is that they can _only_ be
used in templates, which limits the places where they can be used to extract
common functionality

This RFC proposes adding a way to create helpers within JavaScript directly,
extending the reactive model in a way that allows users to extract common
reactive code and patterns, and reuse them transparently. This will make helpers
the new reactive atom of the system - the reactive equivalent of a "function" in
our incremental model. Like components, they have a lifecycle, and can update
over time. Unlike components, they can exist nested in JavaScript classes _and_
in templates, and they can produce any type of value, making them much more
flexible.

```js
import Component from '@glimmer/component';
import { remoteData } from '../helpers/fetch';
import { poll } from '../helpers/remoteData';
import Helper from '@ember/component/helper';

class FetchTask {
  @tracked isLoading = true;
  @tracked isError = false;
  @tracked result = null;

  constructor(url) {
    this.url = url;
    this.run();
  }

  async run() {
    try {
      let response = await fetch(this.url);
      this.result = await response.json();
    } catch {
      this.isError = true;
    } finally {
      this.isLoading = false;
    }
  }

  refresh() {
    this.run();
  }
}

class RemoteData extends Helper {
  compute([url]) {
    return new FetchTask(url);
  }
}


class Poll extends Helper {
  intervalId = null;

  compute([callback, pollPeriod]) {
    cancelInterval(this.intervalId);

    this.intervalId = setInterval(callback, pollPeriod);
  }

  willDestroy() {
    cancelInterval(this.intervalId);
  }
}

export default class Search extends Component {
  data = invokeHelper(this, RemoteData, () => ({
    positional: [`www.example.com/search?query=${this.text}`]
  }));

  constructor() {
    super(...arguments);

    invokeHelper(this, Poll, () => ({
      positional: [() => this.data.value.refresh(), this.pollPeriod]
    }));
  }
}
```

Note how the concerns are completely separated in this version of the component.
The polling logic is self contained, and separated from the data fetching logic.
Both sets of logic are able to contain their lifecycles, updating based on
changes to tracked state, and tearing down when the program destroys them. In
the future, convenience APIs can be added to make invoking them easier to read:

```js
export default class Search extends Component {
  @use data = remoteData(() => `www.example.com/search?query=${this.text}`);

  constructor() {
    super(...arguments);

    use(this, poll(() => [() => this.data.refresh(), this.pollPeriod]));
  }
}
```

## Dependencies

This RFC depends on the [Helper Manager RFC](https://github.com/emberjs/rfcs/pull/625).

## Detailed design

This RFC proposes adding the `invokeHelper` function, imported from
`@ember/helper`. The function will have the following interface (using
TypeScript types for brevity and clarity):

```ts
interface TemplateArgs {
  positional?: unknown[],
  named?: Record<string, unknown>
}

type HelperDefinition = object;

interface Helper {
  value: unknown;
}

function invokeHelper(
  context: object,
  definition: HelperDefinition,
  argsGetter?: (context: object) => TemplateArgs
): Helper;
```

Let's step through the arguments to the function one by one:

#### `context`

This is the parent context for the helper definition. The helper will be
associated as a destroyable to this parent context, using the destroyables API,
so that its lifecycle is tied to the parent. The only requirement of the parent
is that is an object of some kind that can be destroyed.

This allows helper's lifecycles to be entangled correctly with the parent, and
encourages users to ensure they've properly handled the lifecycle of their
helper.

#### `definition`

This is the helper definition. It can be any object, with the only requirement
being that a helper manager has been associated with it via the
`setHelperManager` API.

#### `argsGetter`

This is an optional function that produces the arguments to the helper. The
function receives the parent context as an argument, and must return an object
with a `positional` property that is an array and/or a `named` property that is
an object.

This getter function will be _autotracked_ when it is run, so the process of
retrieving the arguments is autotracked. If any of the values used to create the
arguments object change, the helper will be updated, just like in templates.

### Return Value

The function returns an instance of the helper. The public API of the instance
consists of a `value` property, which will internally be implemented as a getter
that triggers the proper lifecycle hooks on the helper and returns its value, if
it has a value. If it does not, then the helper will do nothing when `value` is
accessed and it will always return `undefined`.

If the helper has a scheduled effect, there is no public API for users to access
the effect or run it eagerly. It will run as scheduled, until the helper is
destroyed.

Using `destroy()` from the destroyables API on the helper instance will trigger
its destruction early. Users can do this to clean up a helper before the parent
context is destroyed.

### Effect Helper Timing Semantics

Standard helpers that return a value will only be updated when they are used,
either in JavaScript or in the template. The args getter and the relevant helper
manager lifecycle hooks will be called when the `value` property on the helper
is used.

Side-effecting helpers, by contrast, run their updates specifically when
scheduled. When introduced by the Helper Manager RFC, there was no relative
ordering specified in the scheduling of side-effecting helpers, because there
was no way for them to have _children_, and we don't generally specify ordering
of siblings. With this RFC, it will be possible to invoke a side-effecting
helper within another side-effecting helper, so they will be able to have
children for the first time.

This RFC proposes modifying the Helper Manager RFC to specify that the
`runEffect` hook of a helper always runs _after_ the `runEffect` hooks of its
children. This mirrors the timing semantics of modifier hooks in templates.

### Ergonomics

This is a low-level API for invoking helpers and creating instances. The API is
meant to be functional, but not particularly readable or ergonomic. This API can
be wrapped with higher level, more ergonomic APIs in the ecosystem, until we're
sure what the final API should be.

## How we teach this

This API is meant to be a low-level primitive which will eventually be replaced
with higher level wrappers, possibly decorators, that will be much easier to use
and recommend to average app developers. As such, it will only be taught through
API documentation.

### API Docs

#### `invokeHelper`

The `invokeHelper` function can be used to create a helper instance in
JavaScript.

```js
import Component from '@glimmer/component';
import Helper from '@ember/component/helper';
import { invokeHelper } from '@ember/helper';

class FetchTask {
  @tracked isLoading = true;
  @tracked isError = false;
  @tracked result = null;

  constructor(url) {
    this.url = url;
    this.run();
  }

  async run() {
    try {
      let response = await fetch(this.url);
      this.result = await response.json();
    } catch {
      this.isError = true;
    } finally {
      this.isLoading = false;
    }
  }
}

class RemoteData extends Helper {
  compute([url]) {
    return new FetchTask(url);
  }
}

export default class Example extends Component {
  data = invokeHelper(this, RemoteData, () => ({
    positional: [`www.example.com/search?query=${this.text}`]
  }));

  get result() {
    return this.data.value.result;
  }
}
```

It receives three arguments:

* `context`: The parent context of the helper. When the parent is torn down and
  removed, the helper will be as well.
* `definition`: The definition of the helper.
* `argsGetter`: An optional function that produces the arguments to the helper.
  The function receives the parent context as an argument, and must return an
  object with a `positional` property that is an array and/or a `named`
  property that is an object.

And it returns a helper instance which has a `value` property. This property
will return the value of the helper, if it has one. If not, it will return
`undefined`.

## Drawbacks

- Additional API surface complexity. There will be additional ways to use
  helpers that we will have to teach users about in general. This is true, but
  given it helps to solve a lot of common problems that users have in Octane it
  should be worthwhile.

- This API is a primitive that is not particularly ergonomic or user friendly,
  but this is part of the point. It gets the job done, and can be built on top
  of to create a better high level API.

## Alternatives

- The [`@use` and Resources RFC](https://github.com/emberjs/rfcs/pull/567)
  proposes a higher level approach to this problem space, but had a number of
  concerns and drawbacks. After much discussion, we decided that it would be
  better to ship the primitives to build something like it in user-space, and
  prove out the ideas in it.
