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

class PlusOneHelper extends Helper {
  compute([number]) {
    return number + 1;
  }
}

export default class PlusOneComponent extends Component {
  plusOne = invokeHelper(this, PlusOneHelper, () => {
    return {
      positional: [this.args.number],
    };
  });
}
```
```hbs
{{this.plusOne.value}}
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
lifeycle hooks, making components the smallest _atom_ for reactive composition.
This presented an issue for composability in general. In the world of functions,
a piece of code can always be broken out into a new function, giving the user
the ability to extract repeated functionality, abstracting common patterns and
concepts and reducing brittleness. For instance, in the following example we
extract several portions of the `myProgram` function to make it clearer what
each section is doing, and isolate its behavior.

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

Since components are the smallest reactive atom, there often is not a way to do this
transparently, since the only portions of the code that are reactive are the
component hooks themselves. This results in related code being spread across
multiple locations, with the user being forced to keep the relationships between
these bits of code in their head at all times, and understand the interactions
between them.

Consider this search component, which updates performs a cancellable fetch
request and updates the document title whenever the search query is updated:

```js
import Component from '@ember/component';
import { fetch, cancelFetch } from 'fetch-with-cancel';

export default class Search extends Component {
  // Args
  query = '';

  @tracked data;

  didReceiveAttrs() {
    // Update the document title
    document.title = `Search Result for "${this.query}"`;

    // Cancel the previous fetch if it's still running
    cancelFetch(this.promise);

    // create a new fetch request, and set the data property to the response
    this.promise = fetch(`www.example.com/search?query=${this.query}`)
      .then((response) => response.json());
      .then((data) => this.data = data);
  }

  willDestroy() {
    cancelFetch(this.promise);
  }
}
```

This component mixes two separate concerns in its lifecycle hooks - fetching the
data, and updating the document title. We can extract these into utility
functions in some isolated cases, but it becomes difficult when functionality
covers multiple parts of the lifecycle, like with the fetch logic here:

```js
import Component from '@ember/component';
import { fetch, cancelFetch } from 'fetch-with-cancel';

function updateDocumentTitle(title) {
  document.title = title;
}

function updateFetch(url, callback, previousPromise) {
  // Cancel the previous fetch if it's still running
  cancelFetch(previousPromise);

  // create a new fetch request, and set the data property to the response
  return fetch(url)
    .then((response) => response.json());
    .then((data) => callback(data));
}

function teardownFetch(promise) {
  cancelFetch(promise);
}

export default class Search extends Component {
  // Args
  query = '';

  @tracked data;

  didReceiveAttrs() {
    updateDocumentTitle(`Search Result for "${this.query}"`)

    this.promise = updateFetch(
      `www.example.com/search?query=${this.query}`,
      (data) => this.data = data,
      this.promise
    );
  }

  willDestroy() {
    teardownFetch(this.promise);
  }
}
```

We can see here that we needed to add two separate helper functions to extract
the data fetching functionality, one to handle updating the fetch, and one to
handle tearing it down, because those different pieces of code need to run at
different portions of the component lifecycle. If we want to reuse these
functions elsewhere, this adds a lot of boilerplate to integrate the functions
in each lifecycle hook.

There are a few alternatives that would allow us to extract this functionality
together.

1. We could use mixins, since they allow us to specify multiple functions and
   mix them into a class. Mixins however introduce a lot of complexity in the
   inheritance hierarchy and are considered an antipattern, so this is not a
   good choice overall.

2. We could extract the functionality out to separate components. Components
   have a contained lifecycle, so they can manage any piece of functionality
   completely in isolation. This works nicely for the document title, but adds
   a lot of complexity for the data fetching, since we need to yield the data
   back out via the template:

    ```js
    // app/components/doc-title.js
    import Component from '@ember/component';

    export default class DocTitle extends Component {
      didReceiveAttrs() {
        document.title = this.title;
      }
    }
    ```
    ```js
    // app/components/fetch-data.js
    import Component from '@ember/component';
    import { fetch, cancelFetch } from 'fetch-with-cancel';

    export default class FetchData extends Component {
      @tracked data;

      didReceiveAttrs() {
        // Cancel the previous fetch if it's still running
        cancelFetch(this.promise);

        // create a new fetch request, and set the data property to the response
        this.promise = fetch(this.url)
          .then((response) => response.json());
          .then((data) => this.data = data);
      }

      willDestroy() {
        cancelFetch(this.promise);
      }
    }
    ```
    ```hbs
    <!-- app/components/fetch-data.hbs -->
    {{yield this.data}}
    ```
    ```hbs
    <!-- app/components/search.hbs -->
    <DocTitle @title='Search Result for "{{@query}}"'/>

    <FetchData @url="www.example.com/search?query={{@query}}" as |data|>
      ...
    </FetchData>
    ```

   This structure is also not ideal because the components aren't being used for
   templating, they're just being used for logic effectively.

3. We could use other template constructs, such as helpers or modifiers. Both
   helpers and modifiers have lifecycles, like components, and can be used to
   contain functionality. Modifiers aren't really a good choice here though,
   because it would require us to add an element that we don't need. So, helpers
   are the better option.

   Helpers work, but like components they require us to move some of our logic
   into the template, even if that isn't really necessary:

    ```js
    // app/helpers/doc-title.js
    import { helper } from '@ember/component/helper';

    export default helper(([title]) => {
      document.title = title;
    });
    ```
    ```js
    // app/helpers/fetch-data.js
    import Helper from '@ember/component/helper';
    import { fetch, cancelFetch } from 'fetch-with-cancel';

    export default class FetchData extends Helper {
      @tracked data;

      compute([url]) {
        if (this._url !== url) {
          this.url = url;

          // Cancel the previous fetch if it's still running
          cancelFetch(this.promise);

          // create a new fetch request, and set the data property to the response
          this.promise = fetch(url)
            .then((response) => response.json());
            .then((data) => this.data = data);
        }

        return this.data;
      }

      willDestroy() {
        cancelFetch(this.promise);
      }
    }
    ```
    ```hbs
    <!-- app/components/search.hbs -->
    {{doc-title 'Search Result for "{{@query}}"'}}

    {{#let (fetch-data "www.example.com/search?query={{@query}}") as |data|}}
      ...
    {{/let}}
    ```

Out of these options, helpers are the closest to what we want - they produce
computed values directly without a template, and with the recent addition of
effect helpers they can be used side-effect to accomplish tasks like setting the
document title. The only downside is that they can only be invoked in templates,
so they require you to design your components around using them in templates
only. This can be difficult to do in many cases, where the data wants to be
accessed to create derived state for instance.

This RFC proposes adding a way to create helpers within JavaScript directly,
extending the reactive model in a way that allows users to extract common
reactive code and patterns, and reuse them transparently. This will make helpers
the new reactive atom of the system - the reactive equivalent of a "function" in
our incremental model. Like components, they have a lifecycle, and can update
over time. Unlike components, they can exist nested in JavaScript classes _and_
in templates, and they can produce any type of value, making them much more
flexible.

```js
// app/components/search.js
import Component from '@ember/component';

export default class Search extends Component {
  data = invokeHelper(this, FetchData, () => {
    return {
      positional: [`www.example.com/search?query=${this.query}`],
    };
  });

  constructor() {
    super(...arguments);

    invokeHelper(this, DocTitle, () => {
      return {
        positional: [`Search Result for "${this.query}"`],
      };
    });
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
  @use data = fetchData(() => `www.example.com/search?query=${this.query}`);

  constructor() {
    super(...arguments);

    use(this, docTitle(() => `Search Result for "${this.query}"`));
  }
}
```

## Detailed design

This RFC proposes adding the `invokeHelper` function, imported from
`@ember/helper`. The function will have the following interface (using
TypeScript types for brevity and clarity):

```ts
interface TemplateArgs {
  positional?: unknown[],
  named?: Record<string, unknown>
}

type HelperDefinition<T = unknown> = object;

function invokeHelper<T = unknown>(
  parentDestroyable: object,
  definition: HelperDefinition<T>,
  computeArgs?: (context: object) => TemplateArgs
): Cache<T>;
```

Let's step through the arguments to the function one by one:

#### `parentDestroyable`

This is the parent for the helper definition. The helper will be associated as a
destroyable to this parent context, using the [destroyables API](https://github.com/emberjs/rfcs/blob/master/text/0580-destroyables.md),
so that its lifecycle is tied to the parent. The only requirement of the parent
is that it is an object of some kind that can be destroyed. If the parent has an
owner, this owner will also be passed to the helper manager that it is invoked on.

This allows helper's lifecycles to be entangled correctly with the parent, and
encourages users to ensure they've properly handled the lifecycle of their
helper.

#### `definition`

This is the helper definition. It can be any object, with the only requirement
being that a helper manager has been associated with it via the
[`setHelperManager` API](https://github.com/emberjs/rfcs/blob/master/text/0625-helper-managers.md#detailed-design).

#### `computeArgs`

This is an optional function that produces the arguments to the helper. The
function receives the parent context as an argument, and must return an object
with a `positional` property that is an array and/or a `named` property that is
an object.

This getter function will be _autotracked_ when it is run, so the process of
retrieving the arguments is autotracked. If any of the values used to create the
arguments object change, the helper will be updated, just like in templates.

### Return Value

The function returns a Cache instance, as defined in the [Autotracking Memoization RFC](https://github.com/emberjs/rfcs/blob/master/text/0615-autotracking-memoization.md#detailed-design).
This cache returns the most recent value of the helper, and will update whenever
the helper updates. Users can access the value using the `getValue` function for
caches.

If the helper has a scheduled effect, using `getValue` on the cache will not run
it eagerly. It will run as scheduled, until the helper is destroyed.

The cache will be also be destroyable, so that using `destroy()` from the
destroyables API on it will trigger its destruction early. Users can do this to
clean up a helper before the parent context is destroyed.

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

Once community addons are built with higher level APIs that are more ergonomic,
we should also add a section in the guides that uses them to demonstrate
techniques for using helpers in JS. This strategy is similar to how modifiers
are documented today.

### API Docs

#### `invokeHelper`

The `invokeHelper` function can be used to create a helper instance in
JavaScript.

```js
// app/components/data-loader.js
import Component from '@glimmer/component';
import Helper from '@ember/component/helper';
import { invokeHelper } from '@ember/helper';

class PlusOneHelper extends Helper {
  compute([num]) {
    return number + 1;
  }
}

export default class PlusOneComponent extends Component {
  plusOne = invokeHelper(this, PlusOneHelper, () => {
    return {
      positional: [this.args.number],
    };
  });
}
```
```hbs
{{this.plusOne.value}}
```

It receives three arguments:

* `context`: The parent context of the helper. When the parent is torn down and
  removed, the helper will be as well.
* `definition`: The definition of the helper.
* `computeArgs`: An optional function that produces the arguments to the helper.
  The function receives the parent context as an argument, and must return an
  object with a `positional` property that is an array and/or a `named`
  property that is an object.

And it returns a Cache instance that contains the most recent value of the
helper. You can access the helper using `getValue()` like any other cache. The
cache is also destroyable, and using the `destroy()` function on it will cause
the helper to be torn down.

Note that using `getValue()` on helpers that have scheduled effects will not
trigger the effect early. Effects will continue to run at their scheduled time.

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
