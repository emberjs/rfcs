- Start Date: 2019-12-15
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/567
- Tracking: (leave this empty)

# `@use` and Resources

## Summary

Adds an API for defining and using _Resources_, which are lifecycle-driven state
managers. Resources can be used to make a number of common tasks much more
ergonomic, such as loading data in components, setting up and updating
subscriptions to external events, and controlling external libraries.

For instance, once could use `@use` and resources to create a `RemoteData`
resource that manages fetching data from a URL, while presenting the state of
fetching in a declarative way, similar to `ember-concurrency`.

```js
export default class SearchResults extends Component {
  @use products = RemoteData.create(this.args.searchUrl);
}
```

```hbs
{{#if this.products.isLoading}}
  <LoadingSpinner/>
{{else}}
  {{#each this.products.data as |product|}}
    <ProductCard @product={{product}} />
  {{/each}}
{{/if}}
```

## Motivation

Ember Octane introduced autotracking, which is a general-purpose reactivity
system that redefines the way that state flows throughout Ember applications.
Native getters, helpers, and modifiers have all been integrated with
autotracking, and collectively these changes have made state flow more
declarative and targeted, and generally have been well received by early
adopters.

One of the pieces of feedback we've received, however, is that there are some
gaps in the this reactivity sytem. Some example use cases include:

- Triggering asynchronous requests based on changing arguments, and consuming
  the results in templates and getters in a natural way.
- Controlling external libraries such as Leaflet and Three.js using components
  and the rendering tree, rather than manual library code.
- More generally, controlling state outside of Ember itself. A great example is
  [TurboPatent](https://turbopatent.com/), an Electron app which needs to sync
  the state of the app with the state of the _native_ menu bar, outside of the
  document and Ember's rendering layer entirely.

These use cases were handled sporadically in a number of different ways in
Ember Classic. Users could use component lifecycle hooks, observers, and
external libraries like `ember-concurrency` and `ember-async-await-helper` to
solve these problems, but there was not a good general solution that fit all of
them.

Fundamentally, the issues come down to mixing _declarative_ code (like getters
and helpers, where tracked properties are used to produce a new result based on
their values) and _imperative_ code (like component lifecycle hooks Ember
Classic, where a the user wants to run some code that does not return a value
whenever something else changes).

This RFC proposes a new primitive, _Resources_, built specifically for this
general problem of mixing imperative and declarative code; one that will
complement the existing Octane programming model, and improve its ergonomics
overall.

### Case Study: Triggering Async

To demonstrate the gap that Resources fill, we can take a look at a fairly
common use case that users of `ember-concurrency` tasks have had: Rerunnning a
task whenever a component argument changes.

In Ember Classic, it would be straightforward to set this up:

```js
export default Component.extend({
  products: task(function*(url) {
    let response = yield fetch(url);
    let data = yield response.json();
    return data;
  }),

  didReceiveAttrs() {
    this.products.perform(this.searchUrl;)
  },
});
```

With Ember Octane, it is possible to get the same behavior using
`ember-concurrency-decorators` and `@ember/render-modifiers`, but it is a bit
more convoluted:

```js
export default class SearchResults extends Component {
  @task
  products*(url) {
    let response = yield fetch(url);
    let data = yield response.json();
    return data;
  }
}
```

```hbs
<div
  {{did-insert (perform this.loadProducts @searchUrl)}}
  {{did-update (perform this.loadProducts @searchUrl)}}
>
</div>
```

We can see that this has a few disadvantages compared to the original approach:

- It requires us to switch contexts between the template and the JS to
  understand the flow of the component.
- It requires us to explicitly list all of the dependencies for a task _twice_.
  This is reminiscent of dependency lists in computed properties and observers,
  and has similar drawbacks in some cases.
- It requires us to add an element to our template, one which potentially does
  nothing at all other than exist to allow us to invoke the modifiers.

This approach does have one advantage over the original - it will only call
perform if `@searchUrl` has actually changed. This would have potentially been a
subtle bug in the original, since it could cause search results to reload if
unrelated args changed. But the extra weight and other caveats make this
solution _feel_ worse overall, as we've discovered through the experience of
Octane's early adopters.

### Introducing `@use` and Resources

There are two core issues here:

1. With Glimmer components, there is no simple way to define behavior that has
   its own self contained lifecycle, independent of a template.
2. Historically, even before Glimmer components, there has been no standard way
   to mix imperative and declarative code, and expose the results naturally.
   `ember-concurrency` comes close, but only solves one specific problem.

The `@use` API aims to solve both of these issues, and to do so in a way that
integrates with autotracking so that users don't have to list out dependencies
constantly in a repetitive manner. The core of the API consists of:

- The `@use` decorator (and the complementary `{{use}}` helper)
- The `Resource` base class (implemented via a `ResourceManager`, similar to
  both components and modifiers).

With these, we can create the `RemoteData` example at the beginning of this RFC
like so:

```js
export class RemoteData extends Resource {
  @tracked data = null;
  @tracked isLoading = false;

  get state() {
    return {
      isLoading: this.isLoading,
      data: this.data,
    };
  }

  async start(url) {
    this.isLoading = true;

    let response = await fetch(url);
    let data = await response.json();

    if (!this.isDestroyed) {
      this.isLoading = false;
      this.data = data;
    }
  }
}
```

And we would use it like so:

```js
export default class SearchResults extends Component {
  @use products = RemoteData.create(this.args.searchUrl);
}
```

```hbs
{{#if this.products.isLoading}}
  <LoadingSpinner/>
{{else}}
  {{#each this.products.data as |product|}}
    <ProductCard @product={{product}} />
  {{/each}}
{{/if}}
```

### A Closer Look

Resources fundamentally revolve around two lifecycle hooks: `start` and
`teardown`.

After a new instance of a `Resource` is created it runs the `start` method,
which receives the arguments passed to `Resource.create()` and is used to
setup the state. This method and its arguments are autotracked, and if anything
that was consumed in it changes, `teardown` is called to destroy the
`Resource`, and a new one is created with the new state in its place. When
the _parent_ of the `Resource` (the class it is `@use`d on) is destroyed,
then its `teardown` hook is called one last time and it is removed completely.

Users can then define the `state` property, which is exposed externally on the
field that was decorated with `@use`.

```js
class Counter extends Resource {
  @tracked count = 0;

  intervalId = null;

  // This is the value that is returned when `this.counter` is accessed on
  // the component, either in the template, or in the JS of the component.
  get state() {
    return this.count;
  }

  // This runs the first time when an instance of MyComponent is created,
  // and then whenever `this.args.interval` changes.
  start(interval) {
    this.intervalId = setInterval(() => this.count++, interval);
  }

  // This whenever `this.args.interval` changes, and when the parent
  // MyComponent is destroyed.
  teardown() {
    clearInterval(this.intervalId);
  }
}

// usage
class MyComponent extends Component {
  @use counter = Counter.create(this.args.interval);

  get counterPlusOne() {
    // `this.counter` is equal to the return value of the `state`
    // getter on Counter.
    return this.counter + 1;
  }
}
```

```hbs
<!-- this shows the return value of the `state` getter on Counter -->
{{this.counter}}
```

If users don't want their `Resource` to be torn down and remade for every
change, they can also add an `update` method to the definition. This will be
called for any changes to consumed state instead, and will be autotracked in the
same way that `start` is. It also receives the updated arguments that `start`
receives, allowing the `Resource` to respond to external state changes.

```js
class Interval extends Resource {
  // private, internal state
  currentFn = null;
  currentInterval = null;
  intervalId = null;

  start(fn, interval) {
    this.currentFn = fn;
    this.currentInterval = interval;

    this.intervalId = setInterval(() => {
      this.currentFn();
    }, this.currentInterval);
  }

  // If `fn` or `interval` ever change, this hook is run again with
  // the new values.
  update(fn, interval) {
    // update the function
    this.currentFn = fn;

    // only update the interval if the time period has changed
    if (interval !== this.currentInterval) {
      clearInterval(this.intervalId);

      this.currentInterval = interval;
      this.intervalId = setInterval(() => {
        this.currentFn();
      }, this.currentInterval);
    }
  }

  teardown() {
    clearInterval(this.intervalId);
  }
}

// usage
class MyComponent extends Component {
  @use greeter = Interval.create(this.sayHello, this.args.interval);

  sayHello() {
    console.log('Hello!');
  }
}
```

This design allows users to mix imperative and declarative code in an ergonomic
way, anywhere that it is needed.

### Stepping Back

Resources are designed to address a gap in the Octane programming model: running
imperative, lifecycle driven code based on changes to state. This gap is
exacerbated by changes in Octane, but it is not entirely new; Ember Classic
never really provided a full answer to these problems, and that's why libraries
like `ember-concurrency` were built in the first place.

The following table shows the different broad categories of state flow in an
Ember app, and shows which features from the Classic edition cover which types
of state flow.

- **DOM Effects/Plugins**: Includes using native APIs to customize the DOM, add
  event listeners, and use other built-in browser APIs, along with setting up
  external JS libraries (e.g. `Tether.js`, animation libraries).
- **Declarative Side-Effects**: Includes using render-less components to control
  external libraries, like Three.js and Leaflet, and to side-effect otherwise
  just by their existence in the app.
- **Self-contained State**: Includes setting up, managing, and exposing some
  sort of state that is not easy to control in a declarative way, such as async
  requests.
- **Derived State**: Includes deriving state from base values (like `@tracked`
  properties) in the app and producing new values directly from them.

> - ✅ means that the feature is _generally_ good at covering this type of state
>  flow without major caveats, though it may still have nuances.
> - ⚠️ means that the feature partially covers this type of state flow, but it has
>  major caveats or use cases which aren't covered well or at all.
> - ⛔ means that the feature does not cover this type of state flow.

<table>
  <thead>
    <tr>
      <th></th>
      <th>DOM Effects/Plugins</th>
      <th>Declarative Side-Effects</th>
      <th>Self-contained State</th>
      <th>Derived State</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Classic Component Lifecycle</td>
      <td>✅</td>
      <td>⚠️[1]</td>
      <td>⛔</td>
      <td>⛔</td>
    </tr>
    <tr>
      <td>Observers</td>
      <td>⛔</td>
      <td>⚠️[2]</td>
      <td>⛔</td>
      <td>⛔</td>
    </tr>
    <tr>
      <td>Ember Concurrency</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>⚠️[3]</td>
      <td>⛔</td>
    </tr>
    <tr>
      <td>Computed Properties</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>✅</td>
    </tr>
    <tr>
      <td>Helpers</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>✅</td>
    </tr>
  </tbody>
</table>

> 1. Works for some types of side-effects nicely, like controlling external UI
>    libraries like Leaflet, but not for all types of side-effects, particularly
>    ones that don't map directly to component usage.
> 2. Works for some types of side-effects, but only ones that can be modeled
>    using chains of dependencies. Introduces lots of complicated data flows,
>    which are difficult to predict and debug.
> 3. Works incredibly well, and has had a major impact on the community, but
>    solves one specific problem (async) and is not very generalizable.

We can see that the classic model had some pretty major gaps, most of which were
_partially_ covered, but not completely. It also wasn't entirely clear when to
use which solution for which problem - e.g. should you use observers or
lifecycle hooks to control an external library?

Resources plug this gap, and do so in a way that is unambiguous - DOM related
tasks are handled by modifiers, other lifecycle driven tasks are handled by
resources, and derived state is handled by getters and helpers. All of these are
fundamentally driven by autotracking based on tracked properties, meaning that
each construct only updates when needed - as opposed to classic lifecycle hooks,
which would run with any changes.

<table>
  <thead>
    <tr>
      <th></th>
      <th>DOM Effects/Plugins</th>
      <th>Declarative Side-Effects</th>
      <th>Self-contained State</th>
      <th>Derived State</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Modifiers</td>
      <td>✅</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>⛔</td>
    </tr>
    <tr>
      <td>Resources</td>
      <td>⛔</td>
      <td>✅</td>
      <td>✅</td>
      <td>⛔</td>
    </tr>
    <tr>
      <td>@tracked + Getters</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>✅</td>
    </tr>
    <tr>
      <td>Helpers</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>⛔</td>
      <td>✅</td>
    </tr>
  </tbody>
</table>

The fact that these concepts are so closely related is not a coincidence - they
are complementary, and their APIs should likely reflect that. It's beyond the
scope of this RFC to specify what the final built in Modifier design is, but it
would make sense for it to match the `Resource` API proposed in this RFC,
along with a new Helper API at some point in the future.

## Detailed design

The design of `@use` consists of four parts:

1. The `ResourceManager` interface
2. The `@use` decorator
3. The `{{use}}` helper
4. The `Resource` base class

### `ResourceManager`

Like most concepts that have been added to Ember recently, resources will be
implemented using a manager that is pluggable under the hood. This will ensure
that the design is flexible in case changes to the API need to be made in the
future, and allow for experimentation with new resource base classes and APIs
in general. The `Resource` base class proposed in this RFC is implemented in
terms of this manager, and could also be broken out into its own separate RFC.

The `ResourceManager` interface has the following signature:

```ts
interface ResourceInstance {
  public state;
}

interface ResourceManager {
  capabilities: ResourceManagerCapabilities;

  startResource(
    parent: unknown,
    def: ResourceDefinition,
    args?: unknown[]
  ): ResourceInstance;

  updateResource(
    parent: unknown,
    definition: ResourceDefinition,
    args?: unknown[]
  ): ResourceInstance;

  teardownResource(parent: unknown, def: ResourceDefinition): void;
}
```

The arguments in these APIs are:

- `parent`: Refers to the parent object that the resource is mounted on.
- `definition`: Refers to the definition handle itself, the value that the
  manager was associated with and looked up on.
- `args`: Refers to the arguments array that was set while creating the resource
  definition, if one was set.

`parent` is exposed to the manager here to facilitate APIs that would require
running within the parent's context, such as generator functions:

```js
class MyComponent extends Component {
  @use filteredModels = Task.create(function*() {
    let response = yield fetch(this.args.url);
    let { models } = yield response.json();

    return filterModels(models, this.args.filters);
  });
}
```

Generators are a powerful syntax, but currently they do not have lexical binding
built in (a la arrow functions). Passing the parent to the manager prevents
having to add additional boilerplate for their usage. The default `Resource`
implementation will not directly expose the parent, however.

Both `startResource` and `updateResource` should return an object that has a `state`
getter on it. This allows `@use` and `{{use}}` to maintain a reference to the
internal state of the resource, and get it again whenever it updates. The timing
semantics of the hooks are as follows:

#### `startResource`

- Called after the parent class that the resource is defined on is created.
- Called asynchronously by default, unless the resource is accessed, in
  which case it is called synchronously before returning its internal state.
- Called _before_ the equivalent asynchronous creation resources of any parent:
  - Components
  - Modifiers
  - Resources

#### `updateResource`

- Called after the arguments to the resource have changed, or after any
  values tracked during the previous `startResource` or `updateResource`
  have changed (unless the `disableAutotracking` capability is set to true).
- Called asynchronously by default, unless the resource is accessed, in
  which case it is called synchronously before returning its internal state.
- Called before the equivalent asynchronous update resources of any parent:
  - Components
  - Modifiers
  - Resources

#### `teardownResource`

- Called asynchronously after the instance of the parent class that the resource
  is defined on is scheduled for destruction.
- Called _before_ the equivalent asynchronous destruction resources of any
  parent:
  - Components
  - Modifiers
  - Resources

Resource managers have the following capabilities and defaults:

```ts
interface ResourceManagerCapabilities {
  disableAutotracking: boolean = false;
}
```

The effects of these capabilities are as follows:

- `disableAutotracking`: If set to false, then the `startResource` and
  `updateResource` hooks will be autotracked, and changes to the state consumed
  while running them will trigger the `updateResource` hook again. If set to
  true, only changes to the incoming arguments themselves will cause updates.

Resource managers can be set on classes or objects using an API similar to the
component and modifier manager APIs:

```js
setResourceManager(
  owner => new CustomResourceManager(owner),
  CustomResource
);
```

Like other managers, this manager will inherit down the prototype hierarchy, so
it only needs to be set on the base class.

### The `@use` Decorator

The `@use` decorator is responsible for capturing the definition of a resource,
getting its manager, and calling the appropriate lifecycle methods on it at the
appropriate time.

The internal implementation of the `@use` decorator will be private API, but
for the purposes of this RFC, it's easier to discuss the design if we can talk
about what `@use` expands to, so we will use the example below to discuss the
details of how it works. This should _not_ be considered to be the final
implementation.

```js
@use myState = Resource.create(this.args.foo);

// could expand to
get myState() {
  let instance = createOrUpdateUse(this, 'myState', () => Resource.create(this.args.foo));

  return instance.state;
}
```

On initial creation, the first time it is run, the class field initializer
should always return a handle that has a `ResourceManager` associated with it
via `setResourceManager`. On subsequent updates, the previous manager that
was used for that same key will be used again, so a different value can be
returned instead.

#### Creating vs Updating

Using initializers with `@use` is very nice for end users, but presents a
problem for authors of `ResourceManager`s - initializers don't receive
arguments, so there is no direct way to change behavior between creating a
resource and updating it. Even if they could, it would also be very unergonomic
to force users to pass around arguments through their code:

```js
@use myState = (owner, isCreating) => isCreating ?
  // creating Resource, return a new instance
  Resource.create(owner, this.args.foo) :
  // updating Resource, use the previous instance
  null
```

Instead, `@use` will be accompanied by a helper function: `getUseInfo`.

```ts
declare function getUseInfo(): {
  owner: Owner;
  isCreating: boolean;
  isUpdating: boolean;
};
```

This information will be populated before calling the initializer, and can be
called _exactly once_ during initialization. Calling a second time will throw an
assertion, which is meant to prevent abuse of the API by other code (e.g. other
classes that are initialized when creating the Resource). For safety, it should
always be called immediately in `DEBUG` mode by any Resource implementation,
even if it is not actually needed. Not calling it at all in `DEBUG` mode will
also throw an assertion to help make sure implementers don't leak it
accidentally to user code.

#### Setting Arguments

Additionally, initializers can only return one value, which is the manager
handle itself. An important aspect of resources is that they should be able to
respond to incoming arguments that are consumed while running the initializer,
and in order to do this, the arguments have to be passed to the manager. In
order to do this, users can use the `setUseArgs` method.

```ts
declare function setUseArgs(...args: unknown[]): void;
```

Like with `getUseInfo`, this function can be used _exactly once_ during the
initializer's execution to save off the arguments, and not using it in DEBUG
mode will throw an assertion to help make sure it isn't accidentally leaked to
user code.

#### Scheduling `@use` Initialization

As mentioned above, the timing semantics of `@use` properties are that they
start automatically, asynchronously, after the parent class they are defined on
with is created, or when they are first consumed, whichever is first. Because of
limitations of Ember's current stage 1/legacy decorators, this is not possible
just by decorating a class element such as a field or method, since we intend to
create a getter and not a field. Getters do not run any code on initialization.

As such, Ember will provide a function that can be used to schedule starting any
`@use` instances defined on the class, `scheduleUseStart()`

```js
class GlimmerComponent {
  constructor() {
    scheduleUseStart(this);
  }
}
```

This method only needs to be called once per instance of a class, and can be
called by parent classes. It is generally intended to be an implementation
detail. The following framework provided classes will use this function by
default:

- `EmberComponent`
- `GlimmerComponent`
- `Helper`
- `Route`
- `Controller`
- `Service`
- `Location`
- `Resource`

In general, the policy from this RFC forward will be that framework provided
classes will use this method by default.

### The `{{use}}` Helper

Resources should also be usable without a component class, in cases where they
don't exist, and for more dynamic use cases. The `{{use}}` helper allows
resources to be setup based solely on the template.

```hbs
{{use SyncDocumentTitle @title}}
```

This helper will create and invoke the resource dynamically, with the same
semantics as `@use`. Since helpers are always consumed, resources used with
`{{use}}` will always be eager. If the resource definition itself were to
change, the previous instance of the last resource is completely torn down, and
a new instance of the new resource is created. The `{{use}}` helper will return
the value of the `state` getter on the resource.

`{{use}}` will not attempt to resolve the resource, instead requiring the actual
class (or other base implementation of the resource) to be passed as the first
argument. This will limit its flexibility initially, since it will likely mean
that the best way to get a value to it will be via a component class.

```js
import SyncDocumentTitle from './resources';

class TitleSync extends Component {
  SyncDocumentTitle = SyncDocumentTitle;
}
```

```hbs
{{use this.SyncDocumentTitle @title}}
```

The expectation is that once template imports are available, this form of
invoking resources will be much more usable.

### `Resource`

The `Resource` base class is a framework provided implementation of a resource
base class using a `ResourceManager`. It has the following signature:

```ts
declare class Resource {
  public state;
  public isDestroying;
  public isDestroyed;

  static create();

  start?(...args);
  update?(...args);
  teardown?();
}
```

#### `state`

A public field that represents the externally visible state of the `Resource`
instance. Can be a normal field or a getter, and accesses to it will be
autotracked like usual.

#### `isDestroying` and `isDestroyed`

Public fields that expose the current state of the modifier, whether it is
destroyed or destroying.

#### `create`

A static method that receives an array of arguments and returns an instance of
the Resource. The arguments passed will be stored and passed later on to
`start` and `update`.

#### `start`

A lifecycle hook that starts the `Resource` and sets up its initial state.

- Runs after a new instance of the `Resource` class has been created.
- Receives the arguments passed to `Resource.create()`.
- By default `start` runs asynchronously, but if the state of the instance is
  consumed it will run synchronously before returning the state.
- Is autotracked, and any changes to state that is consumed while running
  `start` will trigger the `update` method if it exists, and the `teardown` hook
  otherwise.

#### `update`

A lifecycle hook that updates the `Resource` with fresh arguments.

- If defined, runs after the arguments to a `Resource` instance change, or
  after any state tracked in the previous `start` or `update` call has changed.
- Receives fresh versions of the arguments passed to `Resource.create()`.
- By default `update` runs asynchronously, but if the state of the instance is
  consumed it will run synchronously before returning the state.
- Is autotracked, and any changes to state that is consumed while running
  `update` will trigger another `update`.

#### `teardown`

A lifecycle hook that tears the `Resource` instance down.

- If no `update` method is defined, runs on the previous instance of
  `Resource` whenever args or state tracked in `start` updates, before a new
  instance of the `Resource` is created.
- Runs before the parent that the `Resource` belongs to is destroyed.

#### TypeScript Support

TypeScript is not officially supported by Ember, but as its adoption grows it's
a good idea to make sure that the APIs we are designing _can_ be typed,
especially with APIs that use decorators. TypeScript users have historically had
many issues with APIs that use decorators and are similar to `@use`, since
decorators cannot change types currently. TS support for
`ember-concurrency-decorators` for instance, is very difficult.

Since `@use` takes an instance of a class that is assigned to a class field and
instead makes the field return the `state` property of that class, it could face
similar issues here:

```ts
class MyState extends Resource {
  state: number = 123;
}

class MyComponent extends Component {
  // This field will be a readonly number, but if `create` is being accurate,
  // TS will _think_ it is an instance of MyState
  @use myState = MyState.create();
}
```

However, we can fudge the types a bit here to make it work better for TS:

```ts
declare class Resource<T> {
  state: T;

  static create(): T;
}
```

From most user's perspectives, the return value of the static `create` method
should be opaque anyways, and doing this will allow TS to "just work" with
TypeScript out of the box.

### Module API

The APIs described in this RFC will be importable from the `@glimmer/tracking`
package:

```js
import { use } from '@glimmer/tracking';
import Resource, {
  setResourceManager,
  getUseInfo,
  setUseArgs,
  scheduleUseStart,
} from '@glimmer/tracking/resource';
```

### Name Choice

The term "resource" is used in many programming languages to refer to a value
which inherently has a managed lifecycle. Some examples include

* The [RAII](https://en.wikipedia.org/wiki/Resource_acquisition_is_initialization)
  pattern, which the design of resources was heavily influenced by (with the
  addition of an "update").
* The [try-with-resource](https://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html)
  syntax from Java, and the [explicit-resource-management](https://github.com/tc39/proposal-explicit-resource-management)
  proposal for JavaScript.

## How we teach this

`@use` and resources are a new concept in Ember Octane, but they are
fundamentally related to the existing state flow patterns that were established.
They should be introduced in the `Components` section of the guides, after
modifiers. The docs should include useful examples for how to accomplish basic
tasks, such as writing a `RemoteData` resource.

### API Docs

TODO

## Drawbacks

- Adds a new construct to the Octane programming model, which is another concept
  to learn and more overhead overall. However, it reduces a number of ad hoc
  concepts that were filling these gaps anyways.

## Alternatives

- Continue without adding a new construct, and recommend a mixture of helpers
  and modifiers for handling the types of side-effecting code that resources
  are designed for.

- The timing of resources starting and updating is currently dependent on when they
  are consumed. This could be changed to be an explicit `ResourceManager`
  capability, with resournces being _only_ scheduled or _only_ consumed, and without
  forking behavior. In practice, it's unclear when this would matter.

- As mentioned above in the detailed design section, the `Resource` base
  class could be broken out into a separate RFC, and this RFC could focus on
  landing the manager. This strategy worked well for component managers, which
  already had a component class that was in common use, but has had mixed
  reactions with modifiers, and has caused a fair amount of confusion about the
  modifier API, so it may be better to land an initial implementation in the
  first pass, especially given the minimal API surface area.

- One `Resource` base class is proposed in this API, but we could have more
  than one for specific use cases - one for producing state, and another for
  reflecting it, for instance.

- The "resource" terminology conflicts with a few other usages, most notably
  HTTP resources.

  - `LiveState`: The idea being that the class represents self contained state
    that is changing, hence "live". This gets confusing when talking about the
    state of the `LiveState` though, and is hard to pluralize.

  - `Usable`: This term is the opposite of `@use`, and makes it clear they go
    together. The downside is it doesn't really have any meaning on its own.
    Everything is usable, for some definition of the word - what is unique to
    this API?

  - `TrackedHook`: Resources consist of lifecycle hooks, and conceptually they
    occupy a similar space to "hooks" in other frameworks. However,
    fundamentally they are fairly different in the end, and the term could be
    more confusing than helpful because of this. This terminology also makes
    discussing the lifecycle hooks of resources themselves difficult.

- The design of this RFC is heavily based on the limitations of stage 1/legacy
  decorators, in particular the inabliity for a decorator to add a getter _and_
  initializer to the same field. We could update the transform, or `@use`
  specifically, to circumvent this and allow us to run initialization code upon
  the creation of the class. This could be dangerous, since we don't know if the
  final version of decorators will allow this.
