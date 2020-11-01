---
Start Date: 2018-12-05
RFC PR: https://github.com/emberjs/rfcs/pull/410
Relevant Team(s): Ember.js
Authors: Tom Dale, Chris Garrett, Chad Hietala, Yehuda Katz
Tracking: https://github.com/emberjs/rfc-tracking/issues/4

---

# Tracked Properties

## Summary

Tracked properties introduce a simpler and more ergonomic system for tracking
state change in Ember applications. By taking advantage of new JavaScript
features, tracked properties allow Ember to reduce its API surface area while
producing code that is both more intuitive and less error-prone.

This simple example shows a `Person` class with three tracked properties:

```js
export default class Person {
  @tracked firstName = 'Chad';
  @tracked lastName = 'Hietala';

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

### A Note on Decorator Support

This RFC proposes a decorator version of tracked properties, and uses this
decorator version in most examples, on the assumption that the [Decorators RFC]
(https://github.com/emberjs/rfcs/pull/408) will be accepted and implemented
before this RFC. If the Decorators RFC is _not_ accepted, or cannot be
implemented due to other criteria not being met (such as decorators remaining at
stage 2), then only the classic class syntax for tracked properties will be
implemented.

## Terminology

Because of the occasional overlap in terminology when discussing similar
features, this document uses the following language consistently:

- A **getter** is an ES5 JavaScript feature that executes a function to
  determine the value of a property. The function is executed every time the
  property is accessed.
- A **computed property** is a property on an Ember object whose value is lazily
  produced by executing a function. That value is nearly always cached until one
  of computed property's dependencies changes.
- A **tracked property** refers to any class field that has been instrumented
  with `@tracked`. Unlike computed properties, tracked properties are _never_
  getters or setters.
- The **classic programming model** refers to the traditional Ember programming
  model. It includes _classic classes_, _computed properties_, _event
  listeners_, _observers_, _property notifications_, and _classic components_,
  and more generally refers to features that will not be central to Ember
  Octane. Concepts like _routes_, _controllers_, and _services_ belong to both
  the Octane programming model and the classic programming model.
- **Native classes** are classes defined using the Javascript `class` keyword.
- **Classic classes** are classes defined by subclassing from `EmberObject`
  using the static `extend` method.

## Motivation

Tracked properties are designed to be simpler to learn, simpler to write, and
simpler to maintain than today's computed properties. In addition to clearer
code, tracked properties eliminate the most common sources of bugs and mental
model confusion in computed properties today, and reduce memory overhead by not
caching by default.

### Leverage Existing JavaScript Knowledge

Ember's computed properties provide functionality that overlaps with native
JavaScript getters and setters. Because native getters don't provide Ember with
the information it needs to track changes, it's not possible to use them
reliably in templates or in other computed properties.

New learners have to "unlearn" native getters, replacing them with Ember's
computed property system. Unfortunately, this knowledge is not portable to other
applications that don't use Ember that developers may work on in the future, and
while this problem may be lessened by adopting native classes and decorators, it
still requires users learn Ember's notification system and its quirks.

Tracked properties are as thin a layer as possible on top of native JavaScript.
Tracked properties look like normal properties because they _are_ normal
properties.

Because there is no special syntax for retrieving a tracked property, any
JavaScript syntax that feels like it should work does work:

```js
// Dot notation
const fullName = person.fullName;
// Destructuring
const { fullName } = person;
// Bracket notation for computed property names
const fullName = person['fullName'];
```

Similarly, syntax for changing properties works just as well:

```js
// Simple assignment
this.firstName = 'Yehuda';
// Addition assignment (+=)
this.lastName += 'Katz';
// Increment operator
this.age++;
```

This compares favorably with APIs from other libraries, which becomes more
verbose than necessary when JavaScript syntax isn't available:

```js
this.setState({
  age: this.state.age + 1,
});
```

```js
this.setState({
  lastName: this.state.lastName + "Katz";
})
```

### Avoiding Dependency Hell

Currently, Ember requires developers to manually enumerate a computed property's
dependent keys: the list of _other_ properties that _this_ computed property
depends on. Whenever one of the listed properties changes, the computed
property's cache is cleared and any listeners are notified that the computed
property has changed.

In this example, `'firstName'` and `'lastName'` are the dependent keys of the
`fullName` computed property:

```js
import EmberObject, { computed } from '@ember/object';

const Person = EmberObject.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  }),
});
```

While this system typically works well, it comes with its share of drawbacks.

First, it's extra work to have to type every property twice: once as a string as
a dependent key, and again as a property lookup inside the function. While
explicit APIs can often lead to clearer code, this verbosity has the potential
to complicate the implementation without improving developer intent at all.
People understand intuitively that they are typing out dependent keys to help
_Ember_, not other programmers.

It's also not clear what syntax goes inside the dependent key string. In this
simple example it's a property name, but nested dependencies become a property
path, like `'person.firstName'`. (Good luck writing a computed property that
depends on a property with a period in the name.)

You might form the mental model that a JavaScript expression goes inside the
string—until you encounter the `{firstName,lastName}` expansion syntax or the
magic `@each` syntax for array dependencies.

The truth is that dependent key strings are made up of an unintuitive,
unfamiliar microsyntax that you just have to memorize if you want to use Ember
well.

Lastly, it's easy for dependent keys to fall out of sync with the
implementation, leading to difficult-to-detect, difficult-to-troubleshoot bugs.

For example, imagine a new member on our team is assigned a bug where a user's
middle name is not appearing in their profile. Our intrepid developer finds the
problem, and updates `fullName` to include the middle name:

```js
import EmberObject, { computed } from '@ember/object';

const Person = EmberObject.extend({
  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.middleName} ${this.lastName}`;
  }),
});
```

They test their change and it seems to work. Unfortunately, they've just
introduced a subtle bug. If the user's `middleName` were to change, `fullName`
wouldn't update! Maybe this will get caught in a code review, given how simple
the computed property is, but noticing missing dependencies is a challenge even
for experienced Ember developers when the computed property gets more
complicated.

Tracked properties have a feature called _autotrack_, where dependencies are
automatically detected as they are used. This means that as long as all
properties that are dependencies are marked as tracked, they will automatically
be detected:

```js
import { tracked } from '@glimmer/tracking';

class Person {
  @tracked firstName = 'Tom';
  @tracked lastName = 'Dale';

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

Note that getters and setters do _not_ need to be marked as tracked, only the
properties that they access need to. This also allows us to opt out of tracking
entirely, like if we know for instance that a given property is constant and
will never change. In general, the idea is that _mutable_, _watchable_
properties should be marked as tracked, and _immutable_ or _unwatched_
properties should not be.

### Reducing Memory Consumption

By default, computed properties cache their values. This is great when a
computed property has to perform expensive work to produce its value, and that
value gets used over and over again.

But checking, populating, and invalidating this cache comes with its own
overhead. Modern JavaScript VMs can produce highly optimized code, and in many
cases the overhead of caching is greater than the cost of simply recomputing the
value.

Worse, cached computed property values cannot be freed by the garbage collector
until the entire object is freed. Many computed properties are accessed only
once, but because they cache by default, they take up valuable space on the heap
for no benefit.

For example, imagine this component that checks whether the `files` property is
supported in input elements:

```js
import Component from '@ember/component';
import { computed } from '@ember/object';

export default Component.extend({
  inputElement: computed(function() {
    return document.createElement('input');
  }),

  supportsFiles: computed('inputElement', function() {
    return 'files' in this.inputElement;
  }),

  didInsertElement() {
    if (this.supportsFiles) {
      // do something
    } else {
      // do something else
    }
  },
});
```

This component would create and retain an `HTMLInputElement` DOM node for the
lifetime of the component, even though all we really want to cache is the
Boolean value of whether the browser supports the `files` attribute.

Particularly on inexpensive mobile devices, where RAM is limited and often slow, we should
be more conservative about our memory consumption. Tracked properties switch
from an opt-out caching model to opt-in, allowing developers to err on the side
of reduced memory usage, but easily enabling caching (a.k.a. memoization) if a
property shows up as a bottleneck during profiling.

## Prior Art

Tracked properties were first implemented in [Glimmer.js](https://glimmerjs.com/),
and were recently polyfilled with clever usage of `notifyPropertyChange` by
the [sparkles-components](https://github.com/rwjblue/sparkles-component/) addon.
These initial implementations inform the design in this RFC, but differ from it
in some key ways. For instance, both Sparkles's and early versions of Glimmer's
`@tracked` did not have an autotracking stack, and instead relied on explicit
dependency keys. After benchmarking showed that autotracking was a viable
strategy, the API for `@tracked` was updated to what is proposed here.

## Detailed Design

This RFC proposes adding the `tracked` decorator function, used to mark class
fields as tracked:

```ts
const tracked: PropertyDecorator;
```

This new function will be exported from `@glimmer/tracking`. Revisiting our
example from earlier, `@tracked` can be used on native class fields and
getters/setters:

```js
import { tracked } from '@glimmer/tracking';

class Person {
  @tracked firstName = 'Tom';
  @tracked lastName = 'Dale';

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

### Getting Tracked Properties

Tracked properties can be accessed using standard Javascript syntax. From the
user's point of view, there is nothing special about them. This should continue
to work in the future, even if new methods are added for accessing properties,
because tracked properties use native getters under the hood.

```js
let person = new Person();

// Dot notation
const fullName = person.fullName;
// Destructuring
const { fullName } = person;
// Bracket notation for computed property names
const fullName = person['fullName'];
```

### Setting Tracked Properties

Tracked properties can be set using standard Javascript syntax. They use native
setters under the hood, meaning that there is no need for using a setter method
like `set`.

```js
let person = new Person();

// Simple assignment
person.firstName = 'Jen';
// Addition assignment (+=)
person.lastName += 'Weber';
// Increment operator
person.age++;
```

### Autotracking

Tracked properties do not need to specify their dependencies. Under the hood,
this works by utilizing an _autotrack stack_. This stack is a bit of global
state which tracked properties can access. As tracked properties are accessed,
they push themselves onto the stack, and once they have finished running, the
stack contains the full list of all the tracked properties that were accessed
while it was running.

In our first example, with the `Person` class, we can see this in action:

```js
import { tracked } from '@glimmer/tracking';

class Person {
  @tracked firstName = 'Tom';
  @tracked lastName = 'Dale';

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }
}
```

When we create a new instance of `Person`, the tracking system has no knowledge
of the connection between `fullName`, `firstName`, and `lastName`. Now, let's
say we go to render this person's name in a component's template:

```hbs
{{this.person.fullName}}
```

When Glimmer accesses the `fullName` property on person, it creates an
_autotrack stack frame_. As we computed `fullName`, any values that are
decorated with `@tracked` push themselves into this stack frame. Because getters
and setters are pure functions, they will ultimately end up accessing some
tracked properties - in this case, the `fullName` getter accesses the
`firstName` and `lastName` properties, and they push themselves onto the stack
frame.

In this way, Glimmer will know about _all_ properties that were accessed when
calculating any bound value in templates.

> **NOTE:** This does _not_ invalidate a cache like in computed properties.
> Internally, Glimmer checks to see if a value has updated _before calling the
> getter_. If it hasn't, then Glimmer does not rerender the related section of
> the DOM. This is effectively an automatic `shouldComponentUpdate` (at least
> the most common usage) from React.

### Manual Invalidation

In user code, the idea that all mutable properties should be marked as tracked
and that all other properties are effectively immutable works well in isolation.
However, there are cases where users will want to work with code they do _not_
control, such as external library code.

Consider the following example. We have a `simple-timer` library that we've
imported from NPM, and we're trying to wrap it with a `TimerComponent` that
uses it to keep track of how much time has passed:

```js
// simple-timer/index.js
export default class Timer {
  seconds = 0;
  minutes = 0;
  hours = 0;

  listeners = [];

  constructor() {
    setInterval(() => {
      this.seconds++;
      this.minutes = Math.floor(this.seconds / 60);
      this.hours = Math.floor(this.minutes / 60);
      this.notifyTick();
    }, 1000);
  }

  notifyTick() {
    for (let listener of this.listeners) {
      listener(this.seconds);
    }
  }

  onTick(listener) {
    this.listeners.push(listener);
  }
}
```

```js
import Timer from 'simple-timer';
import Component, { tracked } from '@glimmer/tracking';

export default class TimerComponent extends Component {
  @tracked timer = new Timer();

  get currentSeconds() {
    return this.timer.seconds;
  }

  get currentMinutes() {
    return this.timer.minutes;
  }
}
```

Even though we've marked the `timer` property as tracked, the `timer.seconds`
property is untracked, and _it_ is the field that is updated. We can solve this
problem by using the timer library's `onTick` event handler to re-set the field,
invalidating it:

```js
export default class TimerComponent extends Component {
  @tracked timer = new Timer();

  constructor() {
    this.timer.onTick(() => {
      // invalidate the timer field.
      this.timer = this.timer;
    });
  }

  get currentSeconds() {
    return this.timer.seconds;
  }

  get currentMinutes() {
    return this.timer.minutes;
  }
}
```

### Interop with the Classic Programming Model

Tracked properties represent a paradigm shift. They are a completely new system,
fully independent of the classic programming model and based on modern
Javascript features and design, and they will be the _default_ change tracking
system in Ember Octane.

However, existing apps, libraries, and addons will be using the classic
programming model for some time, and experience tells us that these sort of
transitions to new features take a while to settle in the community. To ease
this process and enable gradual adoption, tracked properties will be able to
interoperate with the most commonly used features of the classic model:

- Classic classes
- Computed properties
- `get`/`set` and property notifications
- Observers

#### Classic Classes

The `tracked` decorator function will be usable in classic classes, similar to
`computed`:

```js
import EmberObject from '@ember/object';
import { tracked } from '@glimmer/tracking';

const Person = EmberObject.extend({
  firstName: tracked({ value: 'Tom' }),
  lastName: tracked({ value: 'Dale' }),

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  },
});
```

This form will _not_ be allowed on native classes, and will hard error if it is
attempted. Additionally, default values will be defined on the _prototype_ to
maintain consistency with the classic object model.

This will allow existing libraries to transition incrementally, and add tracked
support minimally where necessary. This also brings the _benefits_ of tracked
to classic classes, including the ability to drop usage of `set`:

```js
// before
let person = Person.create();
person.set('firstName', 'Stefan');
person.set('lastName', 'Penner');

// after
let person = Person.create();
person.firstName = 'Stefan';
person.lastName = 'Penner';
```

Ember's `set` function is nowhere to be seen!

#### Computed Properties

Computed properties will interoperate with tracked properties in both
directions:

* Accessing a computed property from a tracked property will add the computed
  property to its list of depedencies. Whenever the computed property is
  invalidated (i.e. because it or one of its dependencies is updated), the
  tracked property will be invalidated as well.

  ```js
  import { tracked } from '@glimmer/tracking';
  import { set } from '@ember/object';
  import { alias } from '@ember/object/computed';

  class Person {
    @tracked firstName;
    @tracked lastName;

    @alias('title') prefix;

    get fullName() {
      return `${this.prefix} ${this.firstName} ${this.lastName}`;
    }
  }

  let person = new Person();

  person.firstName = 'Tom';
  person.lastName = 'Dale';

  set(person, 'title', 'Mr.');

  person.fullName; // 'Mr. Tom Dale'
  ```

* Accessing a tracked property from a computed property will _also_
  automatically add the tracked property to the list of its dependencies. In
  this way, users will be able to gradually add tracked properties and
  simultaneously reap the benefits of not having to use `set` with computeds,
  and not having to specify dependent keys.

  ```js
  import { computed } from '@ember/object';
  import { tracked } from '@glimmer/tracking';

  class Person {
    firstName;
    lastName;

    @tracked middleName;

    @computed('firstName', 'lastName')
    get fullName() {
      return `${this.firstName} ${this.middleName} ${this.lastName}`;
    }
  }

  let person = new Person();

  set(person, 'firstName', 'Tom');
  set(person, 'lastName', 'Dale');

  person.middleName = 'Tomster';

  person.fullName; // 'Tom Tomster Dale'
  ```

It will still be required to use `set` when updating computed properties and
their dependencies. In the future, this restriction could possibly be relaxed.

#### `get` and `set`

It is common in the classic model to set and consume plain object properties
which are not computed properties, or in any other way special. Ember's `get`
and `set` functions historically allowed this by giving us the ability to
intercept all property changes and watch for mutations.

This presents a problem for tracked properties, particularly because of the
recent change in Ember to enable native Javascript getters to replace `get`.
This change means that we have no way to intercept `get`, and consequently no
way for tracked properties to know whether or not a plain property will later
be updated with `set`.

To demonstrate this case, consider the following service and component:

```js
const Config = Service.extend({
  polling: {
    shouldPoll: false,
    pollInterval: -1,
  },

  init() {
    this._super(...arguments);

    fetch('config/api/url')
      .then(r => r.json())
      .then(polling => set(this, 'polling', polling));
  },
});
```
```js
class SomeComponent extends Component {
  @service config;

  get pollInterval() {
    let { shouldPoll, pollInterval } = this.config.polling;

    return shouldPoll ? pollInterval : -1;
  }
}
```
```hbs
{{this.pollInterval}}
```

Let's walk through the flow here:

1. The `SomeComponent` component is rendered for the first time, instantiating
   the `Config` service (assuming this the first time it has ever been
   accessed). The service's init hook kicks off an async request to get the
   configuration from a remote URl.
2. The `pollInterval` property first accesses the service injection when
   rendered, which is a computed property. The property is detected and added to
   the tracked stack.
3. We then access the plain, undecorated `polling` object. Because it is
   is not tracked and not a computed property, tracked does not know that it
   could update in the future.
4. Sometime later, the async request returns with the configuration object. We
   set it on the service, but because our tracked getter did not know this
   property would update, it does not invalidate.

In order to prevent this from happening, user's will have to use `get` when
accessing any values which may be set with `set`, and are not computed
properties.

```js
class SomeComponent extends Component {
  @service config;

  get pollInterval() {
    let shouldPoll = get(this, 'config.polling.shouldPoll');
    let pollInterval = get(this, 'config.polling.pollInterval');

    return shouldPoll ? pollInterval : -1;
  }
}
```

The reverse, however, is not true - computed properties will be able to add
tracked properties, and listen to dependencies explicitly. In some cases, this
may be preferable, though undecorated getters should be the conventional
standard with the long term goal of removing all explicit dependencies and
computed decorations.

#### Observers

While Ember's observer system has been minimized in recent years, it is still
supported in Ember 3 and used occasionally throughout the ecosystem. Observers
use a fundamentally different system for tracking changes than tracked
properties, but this does not mean that it is impossible for the two systems to
interoperate, and it theory it shouldn't require much effort to maintain such
interoperation or regress performance in any meaningful way.

As such, tracked properties will be made to interoperate with observers so that
whenever a tracked property is set using _any_ valid syntax, observers watching
that key will be fired:

```js
import { tracked } from '@glimmer/tracking';
import { addObserver } from '@ember/object/observers';

class Person {
  constructor() {
    addObserver('firstName', () => {
      console.log('firstName changed!');
    });
  }

  @tracked firstName;
}
```

If in the implementation of this RFC it becomes apparent that there _are_ major
caveats to supporting interop with observers, a followup RFC will be made to
address those caveats and make a decision on whether or not to support observers
with those additional constraints.

### Does this mean I still have to use `get` and `set`?

Yes. As mentioned above, interoperating with legacy code will require using
`get` and `set` to be fully safe. However, even in greenfield applications which
do not need to interoperate with legacy addons or code, there will still be use
cases which are _not_ covered by tracked properties. These use cases are roughly
the same as those that come with native [ES Getters][getters]:

1. Objects that implement `unknownProperty` and `setUnknownProperty`
2. [Ember proxies](https://emberjs.com/api/ember/release/classes/ObjectProxy),
   which use `unknownProperty` and `setUnknownProperty`
3. In general, cases where change tracking should be _dynamic_, where the keys
   that are being tracked are _not_ known in advance and cannot be declared
   using decorators.

`get` and `set` will continue to work (as defined in this RFC) and will be
necessary in many applications for the forseeable future. How long exactly is
an [open question addressed below in the unresolved questions section](#unresolved-questions).

[getters]: https://github.com/emberjs/rfcs/blob/master/text/0281-es5-getters.md#motivation


## How we teach this

There are three different aspects of tracked properties which need to be
considered for the learning story:

1. **General usage.** Which properties should I mark as tracked? How do I
   consume them? How do I trigger changes?
2. **Interop with classic systems.** How do I safely consume tracked properties
   from classic classes and computeds? How do I safely consume classic APIs from
   tracked properties?
3. **Interop with non-Ember systems.** How do I tell my app that something has
   changed in MobX objects, RxJS objects, Redux, etc.

### General Usage

The mental model with tracked properties is that anything _mutable_ that is
public should be tracked. If a value will ever change, and it will or could be
watched externally, it should have the `@tracked` decorator attached to it.

After that, usage should be "Just Javascript". You can safely access values
using any syntax you like, including desctructuring, and you can update values
using standard assignments.

```js
// Dot notation
const fullName = person.fullName;
// Destructuring
const { fullName } = person;
// Bracket notation for computed property names
const fullName = person['fullName'];

// Simple assignment
this.firstName = 'Yehuda';
// Addition assignment (+=)
this.lastName += 'Katz';
// Increment operator
this.age++;
```

#### Triggering Updates on Complex Objects

There may be cases where users want to update values in complex, untracked
objects such as arrays or POJOs. `@tracked` will only be usable with class
syntax at first, and while it may make sense to formalize these objects into
tracked classes in some cases, this will not always be the case.

To do this, users can re-set a tracked value directly after its inner values
have been updated.

```js
class SomeComponent extends Component {
  @tracked items = [];

  @action
  pushItem(item) {
    let { items } = this;

    items.push(item);

    this.items = items;
  }
}
```

This may seem a bit strange at first, but it allows users to mentally scope
off a tree of objects. They manipulate internals as they see fit, and the only
operation they need to do to update state is set the nearest tracked property.

### Interop with Classic Systems

There are two cases that we need to consider when teaching interoperability:

1. Accessing non-tracked properties and computeds from an autotrack context
2. Accessing tracked properties from a computed context

In the first case, the general rule of thumb is to use `get` if you want to be
100% safe. In cases where you are certain that the values you are accessing are
tracked, computeds, or immutable, you can safely use standard access syntax.

In the second case, no additional changes need to be made when using tracked
properties. They can be accessed as normal, and will be automatically added to
the computed's dependencies. There is no need to use `get`, and you can use
standard assignments when updating them.

### Interop with Non-Ember Systems

The strategy for trickier updates on complex objects by retriggering their
setters should cover most integration use cases. We should add a guide which
specifically demonstrates their usage by wrapping a common, simple external
library such as `moment.js`. This will demonstrate its usage concretely, and
establish best practices.

## Drawbacks

Like any technical design, tracked properties must make tradeoffs to balance
performance, simplicity, and usability. Tracked properties make a different set
of tradeoffs than today's computed properties.

This means tracked properties come with edge cases or "gotchas" that don't exist
in computed properties. When evaluating the following drawbacks, please consider
the two features in their totality, including computed property gotchas you have
learned to work around.

In particular, please try to compensate for [familiarity][familiarity] and
[loss aversion][loss-aversion] biases. Before you form a strong opinion, [give
it five minutes][5-minutes].

[familiarity]: https://en.wikipedia.org/wiki/Familiarity_heuristic
[loss-aversion]: https://en.wikipedia.org/wiki/Loss_aversion
[5-minutes]: https://signalvnoise.com/posts/3124-give-it-five-minutes

### Tracked Properties & Promises

Dependency autotracking requires that tracked getters access their dependencies
synchronously. Any access that happens asynchronously will not be detected as a
dependency.

This is most commonly encountered when trying to return a `Promise` from a
tracked getter. Here's an example that would "work" but would never update if
`firstName` or `lastName` change:

```js
class Person {
  @tracked firstName;
  @tracked lastName;

  get fullNameAsync() {
    return this.reloadUser().then(() => {
      return `${this.firstName} ${this.lastName}`;
    });
  }

  async reloadUser() {
    const response = await fetch('https://example.com/user.json');
    const { firstName, lastName } = await response.json();
    this.firstName = firstName;
    this.lastName = lastName;
  }

  setFirstName(firstName) {
    // This should cause `fullNameAsync` to update, but doesn't, because
    // firstName was not detected as a dependency.
    this.firstName = firstName;
  }
}
```

One way you could address this is to ensure that any dependencies are consumed
synchronously:

```js
get fullNameAsync() {
  // Consume firstName and lastName so they are detected as dependencies.
  let { firstName, lastName } = this;

  return this.reloadUser().then(() => {
    // Fetch firstName and lastName again now that they may have been updated
    let { firstName, lastName } = this;
    return `${firstName} ${lastName}`;
  });
}
```

However, **modeling async behavior as tracked properties is an incoherent
approach and should be discouraged**. Tracked properties are intended to hold
simple state, or to derive state from data that is available synchronously.

But asynchrony is a fact of life in web applications, so how should we deal with
async data fetching?

**In keeping with Data Down, Actions Up, async behavior should be modeled as
methods that set tracked properties once the behavior is complete.**

Async behavior should be explicit, not a side-effect of property access. Today's
computed properties that rely on caching to only perform async behavior when a
dependency changes are effectively reintroducing observers into the programming
model via a side channel.

A better approach is to call a method to perform the async data fetching, then
set one or more tracked properties once the data has loaded. We can refactor the
above example back to a synchronous `fullName` tracked property:

```js
class Person {
  @tracked firstName;
  @tracked lastName;

  get fullName() {
    return `${this.firstName} ${this.lastName}`;
  }

  async reloadUser() {
    const response = await fetch('https://example.com/user.json');
    const { firstName, lastName } = await response.json();
    this.firstName = firstName;
    this.lastName = lastName;
  }
}
```

Now, `reloadUser()` must be called explicitly, rather than being run implicitly
as a side-effect of consuming `fullName`.

### Accidental Untracked Properties

One of the design principles of tracked properties is that they are only
required for state that _changes over time_. Because tracked properties imply
some overhead over an untracked property (however small), we only want to pay
that cost for properties that actually change.

However, an obvious failure mode is that some property _does_ change over time,
but the user simply forgets to annotate that property as `@tracked`. This will
cause frustrating-to-diagnose bugs where the DOM doesn't update in response to
property changes.

Fortunately, we have a strategy for mitigating some of this frustration. It
involves the way most tracked properties will be consumed: via a component
template. In development mode, we can detect when an untracked property is used
in a template and install a setter that causes an exception to be thrown if it
is ever mutated. (This is similar to today's "mandatory setter" that causes an
exception to be thrown if a watched property is set without going through
`set()`.)

Unfortunately this strategy cannot be applied to values accessed by tracked
getters. The only way we could detect such access would be with native
[Proxies][proxy], but proxies are more focussed on security over flexibility
and recent discussion shows that [they may break entirely when used with
private fields](https://github.com/tc39/proposal-class-fields/issues/106). As
such, it would not be ideal for us to use them in this way.

## Alternatives

### Ship tracked properties in user-land

Instead of shipping `@tracked` today, we can focus on formalizing the primitives
which it uses under the hood in Glimmer VM (References and Validators) and make
these publicly consumable. This way, users will be able to implement tracked in
an addon and experiment with it before it becomes a core part of Ember.

This approach is similar to the approach taken with component managers in the
past year, which unblocked experimentation with `SparklesComponent`s as a way to
validate the design of `GlimmerComponent`s, and unlocked the ability for power
users to create their own component APIs. However, the reference and validator
system is a much more core part of the Glimmer VM, and it could take much longer
to figure out the best and safest way to do this without exposing too much of
the internals. It would certainly prevent `@tracked` from shipping with Ember
Octane.

### Keep the current system

We could keep the current computed property based system, and refactor it
internally to use references only and not rely on chains or the old property
notification system. This would be difficult, since CPs are very intertwined
with property events as are their dependencies. It would also mean we wouldn't
get the DX benefits of cleaner syntax, and the performance benefits of opt-in
change tracking and caching.

### We could keep `set`

Tracked properties were designed around wanting to use native setters to update
state. If we remove that constraint and keep `set`, it opens up some
possibilities. There is precedent for this in other frameworks, such as React's
`setState`.

However, keeping `set` likely wouldn't be able to restrict the requirement for
`@tracked` being applied to all mutable properties for the same reason `get`
must be used in interop - there's no way for a tracked property to know that a
plain, undecorated property could update in the future.

### Allow explicit dependencies

We could allow `@tracked` to receive explicit dependencies instead of forcing
`get` usage for interop. This would be very complex, if even possible, and is
ultimately not functionality `@tracked` should have in the long run, so it would
not make sense to add it now.

### We could wait on private fields and Proxy developments

Native [Proxies][proxy] represent a lot of possibilities for automatic change
tracking. Other frameworks such as Vue and Aurelia are looking into using
recursive proxy structures to wrap objects and intercept access, which would
allow them to track changes without _any_ decoration. We also considered using
recursive proxies in earlier drafts of this proposal, even though they aren't
part of our support matrix we believed they could be used during development to
assert when users attempted to update untracked properties which had been
consumed from tracked getters.

However, as mention above, TC39 has made it clear that this was [not an intended
use for Proxy](https://github.com/tc39/proposal-class-fields/issues/106), and
they will be _breaking_ this functionality with the inclusion of private fields.
They have also expressed that [they would like to solve this
use-case](https://github.com/tc39/proposal-class-fields/issues/162#issuecomment-441101578)
(observing object state changes in general) separately, and [a strawman proposal
was made](https://github.com/littledan/proposal-proxy-transparent) (though it
has not advanced and does not seem like it will). We could wait to see what the
future looks like here, and see if we can provide a more ergonomic tracked
properties RFC in the future.

## Unresolved questions

### When can I stop using `get` and `set`?

This is the biggest open question in this RFC, and with the direction that
tracked properties set. How do we get rid of `get` and `set` for good, if that
is the direction we want to go in?

The full answer to that question is out of scope for tracked properties, but it
would likely require at least two additional steps:

1. The underlying system for tracking changes, including the ability to create
   tags for fields and the ability to add to the current autotracking stack,
   will need to be made public for advanced users who need dynamic change
   tracking.

2. First class support for [native proxies][proxy] within Ember.
   `unknownProperty` and `setUnknownProperty` have no other analag in native
   Javascript, and without support for native proxies there will likely be use
   cases that cannot be supported in any other way.

   As mentioned above, native proxies _will_ (potentially) have more limitations
   than Ember proxies, but these limitations will most likely be possible to
   work around for advanced users who need this functionality in the first
   place. In other words, while they probably don't make sense as a basis for
   _all_ change tracking in Ember, they will probably be invaluable for specific
   use cases such as [Ember M3](https://www.npmjs.com/package/ember-m3) which
   require very dynamic change tracking.

[proxy]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy
