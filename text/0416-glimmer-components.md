---
Start Date: 2018-12-13
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/416
Tracking: https://github.com/emberjs/rfc-tracking/issues/2

---

# Glimmer Components

## Summary

Glimmer components are a simpler, more ergonomic, and more declarative approach
to building components. They represent the sum of multiple years of design and
feature work by the community, which stemmed from the original RFCs and
discussions surrounding "angle-bracket components".

This RFC proposes adding Glimmer components to Ember's public API, and making
them the default new app experience in Ember Octane.

## Attribution

The Glimmer components API presented in this RFC was designed in cooperation
between @tomdale, @rwjblue, @krisselden, @pzuraq, and others.

## First, a Bit of History

As components became the standard for Single Page Apps (SPAs) several years ago,
and the Ember community began adopting them in earnest and converting from Ember
1's primarily MVC oriented approach, there were many small and large issues that
cropped up with Ember's component API and usage: `{{curly-bracket}}` syntax felt
dated with the introduction of Web Components and other major frameworks; the
inability to specify or override HTML attributes led to explosions in API
complexity; the implicit wrapper element and customization fields (`tagName`,
`classNames`, et al.) felt burdensome and made templates difficult to read
compared to other frameworks; two way data-binding led to strange, hard to
predict data cycles within apps; and so on.

From this came the original [Angle Bracket component
RFC](https://github.com/emberjs/rfcs/blob/0b32059b3704836e52c906a0ead64ac186c844d8/active/0000-component-unification.md).
The idea was simple: switch to the superior `<angle-bracket>` syntax of web
components, and solve all the other problems! Seems easy enough, right?

As you can imagine, it was _not_ that easy. There was a flurry of discussion on
the original RFC, and many ideas were thrown around. This was seen as the _one_
chance Ember had to "fix" its component API, and the community did not want to
get it wrong and lock us into yet _another_ set of painful papercuts. After much
debate and lots of back and forth with the design, it was ultimately decided
that attempting to redesign components all at once, monolithically, was too
much. Instead, the individual ideas from that discussion could be broken out and
implemented in isolation, in a backwards compatible way, both incrementally
building a new, well thought out component API _and_ laying the groundwork in
the framework for future redesigns.

A lot of the foundational work that arose from these discussions and paves the
way for Glimmer components has already landed in Ember, including: Angle-bracket
invocation, named arguments, element modifiers, and component managers.

Glimmer components represent the final piece of that's required to enable the
"ember octane" programming model. They include the last of the major features
that were discussed during the original Angle Brackets RFC, and holistically, we
feel those features make a much simpler and more ergonomic component API. Taken
alone, they are an incremental change. Their individual features aren't _that_
much more than what we currently have in Ember today. But as a whole they
represent the culmination of multiple years of design work and discussion by the
Ember community, and the collective attention to detail and care of all of our
community members.

## Terminology

* The **Glimmer VM** is the underlying rendering engine which is used by
  _Ember.js_ and _Glimmer.js_.
* **Glimmer.js** is a thin wrapper on top of the _Glimmer VM_ which exposes a
  much simpler API compared to Ember. Historically it has been used to
  experiment with ideas and implementations before bringing them into Ember via
  RFC, and has been used to write applications which don't require the full
  feature set of Ember.
* **Glimmer components** are a newly proposed component API which draw from the
  experimental APIs provided in Glimmer.js, and Ember.js via
  [sparkles-component](https://github.com/rwjblue/sparkles-component).
* **Classic components** refer to the standard component API at the time of this
  RFC, which have been available in Ember in some form since v1.
* **Tracked properties** refer to a new method of change tracking which is being
  proposed in a separate RFC, parallel to this one.

## Motivation

`GlimmerComponent` is a simpler base component class that enables smaller class
definitions, stronger conventions for lifecycle hooks and properties, and
unidirectional data flow. We aim to design them to be easier understand in
isolation, and require less knowledge of the framework to use effectively.

This example shows a component written with the classic component API:

```hbs
<!-- templates/components/post.hbs -->
{{#if (eq type 'image'}}
  <img src={{post.imageUrl}} title={{post.imageTitle}}>
{{/if}}

{{post.text}}
```
```js
// components/post.js
export default Component.extend({
  tagName: 'section',
  classNames: ['post'],
  classNameBindings: ['type'],
  ariaRole: 'region',

  /* Arguments */
  post: null,

  type: readOnly('post.type'),

  didInsertElement() {
    this._super(...arguments);
    if (this.type === 'image') {
      setupImageOverlay(this.element.querySelector('img'));
    }
  }
});
```

And here is an equivalent component written with the Glimmer component API:

```hbs
<!-- templates/components/post.hbs -->
<section ...attributes role="region" type={{@post.type}} class="post {{@post.type}}">
  {{#if (eq @post.type 'image')}}
    <img
      {{did-insert this.didInsertImage}}
      src={{@post.imageUrl}}
      title={{@post.imageTitle}}
    />
  {{/if}}

  {{@post.text}}
</section>
```
```js
// components/post.js
export default class PostComponent extends GlimmerComponent {
  @action
  didInsertImage(element) {
    setupImageOverlay(element);
  }
}
```

Glimmer components eliminate many of the common paper cuts that cause confusion
with classic components, and align more closely with modern template syntax and
features.

### Outer HTML Semantics

The biggest change Glimmer components make is defaulting to outer HTML
semantics. In the classic component API, components had a implicit wrapper
element. Given this component template:

```hbs
Hello, world!
```

The output by _default_ would be something like:

```html
<!-- OUTPUT -->
<div id="ember-1234" class="ember-view">
  Hello, world!
</div>
```

But we can't know that for sure unless we look at the component definition. If
we do, we might see that the outer wrapping element is actually a `section`, and
it has a `.hello-world` class:

```js
export default Component.extend({
  tagName: 'section',
  classNames: ['hello-world']
});
```
```html
<!-- OUTPUT -->
<section id="ember-1234" class="hello-world ember-view">
  Hello, world!
</section>
```

This behavior means that the template for a component is missing crucial
information and context. Even for the simplest component, users must check the
class definition to know with certainty what the full template of the component
is. And unlike bindings, there is no hint to the user that there may be
something dynamic that they should check on - without advanced knowledge of
Ember's APIs, there is no way of knowing about this behavior.

By contrast, Glimmer components have no wrapping outer element - What you see in
the template is what you get in the output. There is no need to define class
names, class name bindings, attribute bindings, or any other DOM element values
_from the component class_; developers can achieve the equivalent result using
the same techniques they're familiar with from working with `Ember.Component`
templates. The template is the single source of truth for the output of a
component, and any dynamic values are explicitly stated in it.

```hbs
<!-- template.hbs -->
<section class="hello-world">
  Hello, world!
</section>
```

```html
<!-- OUTPUT -->
<section class="hello-world">
  Hello, world!
</section>
```

We can immediately see that this is a simple component with no bindings, no
dynamic values, and no meaningful state. Even if there was a component
definition, we know that it is not in any way affecting the output of this
template. Special element ids and classes are also not present, making the
output appear less magical.

This micro change makes a macro difference:

* Users can spend less time switching back and forth between reading template
  and class code, and can get a better idea of the structure of an app from its
  declarative templates.

* Component customization code becomes less imperative and more declarative,
  meaning users no longer need to keep the state of bindings, class names, and
  other class code in their heads.

* The gap between template-only components - which are analogous to React and
  other frameworks' functional components - and components with a backing class
  is reduced, making them a more viable pattern.

### Namespaced Arguments

In classic components, arguments are set as properties directly on the class
instance. This means that class methods and properties can be completely
overwritten by incoming arguments, which can have surprising and problematic
side effects. For example, let's say we have a component that has a `fullName`
computed property and expects `firstName` and `lastName` arguments:

```js
// components/person.js
export default Component.extend({
  firstName: null,
  lastName: null,

  fullName: computed('firstName', 'lastName', function() {
    return `${this.firstName} ${this.lastName}`;
  })
});
```

The public API of this component is _supposed_ to just be those two arguements.
However, a developer may realize that they can pass `fullName` directly to the
component, overriding the computed property:

```hbs
<Person @fullName={{this.fullNameWithMiddle}}>
```

This is clearly a bad pattern, but it shows that in effect that most details of
a component's implementation are not truly private, and any property or value
can be overriden from external contexts. In most common day-to-day scenarios
developers just have to be careful that they are following the intended public
API of a component, but this also has the potential for misuse and enables
antipatterns.

Glimmer components assign their arguments to the `args` property on their
instance, preventing namespace collisions from happening in the first place.
This allows component authors to define a clear public API for a component which
_cannot_ be circumvented.

### Immutable Arguments

In classic components argument values on the component are also _mutable_. This
can lead to some confusing behavior, because argument values in the _class_ can
change, but named arguments in _templates_ cannot. For instance, given this
component:

```js
// components/welcome.js
export default Component.extend({
  firstName: 'Jen',
  lastName: 'Weber'
});
```
```hbs
<!-- templates/components/welcome.hbs  -->
Hey there, {{@firstName}} {{@lastName}}!
```

And this invocation:

```hbs
<Welcome />
```

You might expect that the result would be:

```hbs
Hey there, Jen Weber!
```

However, `{{@firstName}}` and `{{@lastName}}` would actually be empty values.
They refer _directly_ to the arguments passed into the _invocation_ of the
component, and to get the results we wanted, we would have to invoke it like so:

```hbs
<Welcome @firstName="Jen" @lastName="Weber" />
```

The advantage in this is that named arguments are fully transparent. When seen
in a template, users can know without a doubt that the named argument was a
value passed in from the invocation. Likewise, when they see a standard binding
to a value like `{{this.firstName}}` or `{{firstName}}`, they know this is a
value defined on the component - it could a computed property, it could come
from a service, it could be a random value, but it is _not_ an argument.

Glimmer components align their argument access with named args by making
arguments available exclusively on a (shallow) frozen object, `this.args`.
Attempting to modify `this.args` will hard error, meaning that like templates,
users will always be able to refer to `this.args` as the canonical state of the
values passed to the component invocation.

Immutable arguments make reasoning about the state of a component simpler ("Was
that a user provided value or a default/mutated/computed value?" becomes "Was it
an argument or not?"), and encourages use of `{{@arg}}` syntax in templates
where appropriate. At scale, this makes reading templates even easier, since
more information is encoded in the template itself. One way data flow also
encourages the Data Down, Actions Up pattern, and normalizes the way that data
flows through apps, making reasoning about app state easier.

### Minimal Classes

Classic components are large classes, with lots of built up functionality and
debt from over the years. The total list of default properties and hooks
(including inherited ones) includes:

* **13** Standard lifecycle hooks, such as
  `didInsertElement`/`willDestroyElement` and `didUpdate`.
* **29** Event handlers, such as `click`, `mouseEnter`, and `dragStart`.
* **9** element/element customization properties, such as `element` and
  `tagName`.
* **21** standard framework functions, such as `get`/`set`,
  `addObserver`/`removeObserver` and `toggleProperty`.

Coming from a class hierarchy that is 4 levels deep (`Component` -> `CoreView`
-> `EmberObject` -> `CoreObject`, with about 19 mixins included along the way).

This is a large API surface to become acquainted with, and namespace
collisions are possible with new Ember users - collisions on `destroy`
were the original reason for adding the `actions` object to classes, and every
so often a user will pop in on `#help` wondering why their `click` or `submit`
methods trigger automagically, or why their component disappeared when they
added an `isVisible` property. Even putting aside the possibility of collisions,
the sheer amount of choice can sometimes be overwhelming: Do I put my
initialization logic in `init` or `didInsertElement`? Do I use an action or the
`click` handler? Which update method should I use - `didRender`, `didUpdate`,
`didReceiveAttrs`?

Glimmer components have a constructor, **2** lifecycle hooks, and **3**
properties. They only extend from the Glimmer Component base class -- a simple
ES6 class that does not extend from `EmberObject`. They don't have any
element/DOM based properties, hooks, event handler functions, whose
responsibilities have been passed on to element modifiers. This _dramatically_
simplifies what users need to learn in order to start using the bread-and-butter
class of Ember, and enforces a single conventional location for each of the
possible hooks in classic components, allowing users to focus on productivity
out of the box.

### Glimmer.js Compatibility

One of the goals for future versions of Ember, post Ember Octane, will be to
enable lighter-weight applications to be built using the framework. Breaking
Ember apart into smaller, fully independent and optional pieces is the core idea
behind the "install your way to Ember" goal, which will enable Ember to be used
in more constrained environments that smaller frameworks such as React, Preact,
Vue, and more excel in. It will also allow users who are size-conscious to adopt
Ember incrementally, adding functionality when it is _needed_ rather than having
all-or-nothing.

This will take time though. Progress has been made, but parts of Ember are still
monolithic. And while it isn't Ember, Glimmer.js is a lightweight wrapper of the
Glimmer VM that enables users to drop that weight and begin writing much more
minimal apps today.

Glimmer components aren't just based on the Glimmer.js component API - they are
one and the same. They will be a shared package, which will be importable and
usable by the users of both frameworks. Not only will users be able to write
better components in Ember, those components will also be cross-compatible with
Glimmer apps (assuming they don't use Ember specific functionality).

It is important to note that while Glimmer components will be versioned
independently from Glimmer and Ember ***they will abide by the Ember RFC process
for any and all changes to user APIs***. The implementations for their component
managers in Glimmer and Ember may change to keep them compatible, but they will
not make major changes without first getting community input, and will be
considered part of the public API of Ember.

## Prior Art

Part of the challenge in the original Angle Bracket components RFC was
attempting to design without implementation, testing, usage, and feedback.
Glimmer.js provided an early method to experiment, but because it was not widely
adopted there wasn't much feedback from larger-scale usage. This in part
motivated the [component manager
RFC](https://github.com/emberjs/rfcs/blob/master/text/0213-custom-components.md),
which enabled experimentation in Ember directly, and set us up for having
multiple implementations of component APIs which were interchangeable.

As such, we now have two reference implementations which can be referred to:

* [Glimmer.js](https://glimmerjs.com/), the framework that this component API is
  based on, and will be cross-compatible with.
* [sparkles-component](https://github.com/rwjblue/sparkles-component), an an
  implementation of the Glimmer.js component API using Ember's component
  managers. It is usable in Ember today.

Both of these have minor differences from the API proposed in this RFC, mainly
because they were made before the [element modifier manager
RFC](https://github.com/emberjs/rfcs/blob/master/text/0373-Element-Modifier-Managers.md)
was accepted and opened up additional design possibilities. However, they serve
as valuable data points for the viability of a simpler component API, and inform
the design accordingly.

## Detailed design

Glimmer components have the following interface:

```ts
interface GlimmerComponent<T = object> {
  args: T;

  isDestroying: boolean;
  isDestroyed: boolean;

  constructor(owner: Opaque, args: T): void;
  willDestroy(): void;
}
```

This class will be importable from `@glimmer/component`;

```js
import Component from '@glimmer/component';
```

### Constructor

The constructor for Glimmer components receives two arguments: The owner
instance and the named arguments object. Both of these arguments should
conventionally be passed to `super` immediately, and then accessed through
decorated service properties, `getOwner`, and `this.args`:

```js
class PersonComponent extends GlimmerComponent {
  @service profile;

  constructor() {
    super(...arguments);

    let owner = getOwner(this);
    let profileService = this.profile;

    let firstName = this.args.firstName;
  }
}
```

These arguments are passed to the constructor so that they can be used for
initial setup of the component. Service injections and args being available in
this way also makes them available to class field initializers, which run
immediately _after_ the call to `super`:

```js
class PersonComponent extends GlimmerComponent {
  @service time;

  // Use the values of args in an initializer
  fullName = `${this.args.firstName} ${this.args.lastName}`;

  // Access a service in an initializer
  currentTime = this.time.now();
}
```

The `args` argument will be shallow-frozen (in development mode only) to prevent
users from modifying them.

#### Type Injections

Ember's dependency injection system allows defining injections across an entire
type via
[RegistryProxy#inject](https://www.emberjs.com/api/ember/3.5/classes/ApplicationInstance/methods/inject?anchor=inject) - for
instance, Ember Data's store, which is available by default as
`this.store` on all Routes and Controllers. These injections add a layer of
implicit state to objects, since users must know what the default injections are
ahead of time.

By contrast, service getters (decorated with `@service`) clearly and explicitly
state the dependencies of a class within its definition. The benefit of having
an explicit dependencies list within each class has proven to be invaluable in
practice since `inject.service()` was introduced.

Glimmer components will only receive the owner directly, and as such will _not_
support type injections. This cuts down on the implicit knowledge developers
must have when writing a component.

### Properties

Glimmer components have 3 properties: `args`, `isDestroying`, and `isDestroyed`.

#### `args`

As discussed in the motivation section, `args` is an object with the values of
the named arguments passed to the component. This property will be updated
whenever the arguments change. It will be shallow-frozen in development mode to
prevent users from setting values on it.

#### `isDestroying`

This property will be set to `true` when component teardown has been initiated,
_before_ the component's `willDestroy` hook is run, along with any other
components which are currently being torn down. This allows the entire component
tree to be marked before user code is run. It can be used by users to
conditionally prevent asynchronous code from running, and to check on the
teardown state of the component in general.

#### `isDestroyed`

This property will be set after any `willDestroy` hooks have run, and the
component has been fully torn down. It can be used by users to conditionally
prevent asynchronous code from running, and to check on the teardown state of
the component in general.

### Lifecycle Hooks

Classic components have 13 major lifecycle hooks that run during 3 major phases
of the component lifecycle, with some hooks running during multiple phases:

1. **Initialization and Initial Render:**
   * `init`
   * `willInsertElement`
   * `didInsertElement`,
   * `didReceiveAttrs`
   * `willRender`
   * `didRender`.

2. **Rerenders and Updates:**
   * `didReceiveAttrs`
   * `didUpdateAttrs`
   * `willUpdate`
   * `didUpdate`
   * `willRender`
   * `didRender`

3. **Destruction:**
   * `willDestroyElement`
   * `didDestroyElement`
   * `destroy`
   * `willDestroy`

Many of these hooks have overlapping or redundant functionality, and it's fairly
confusing when to use which and what the differences are. We can simplify this
cycle in a number of ways:

* Hooks that run during multiple phases such as `didRender` and
  `didRecieveAttrs` can be convenient at times, but also add mental overhead and
  redundancy. We can remove these in favor of clearly delineated hooks which
  only run during _one_ phase.

* "Bookend" methods (`did*` and `will*`) can be confusing, since they require
  some specific knowledge of what the "bookended" functionality is. For
  instance, users almost _always_ want to use `didInsertElement` and
  `willDestroyElement`, but the existence of their opposite bookends can make
  this confusing. Additionally, the fact that `didReceiveAttrs` and
  `didUpdateAttrs` do _not_ have opposing bookends is inconsistent with this
  pattern.

* Hooks that are used to update derived state, such as `didUpdate` and
  `didUpdateAttrs`, can be generally be replaced with tracked or computed
  properties that pull the required values as they are used, rather than eagerly
  as they are updated. This is more inline with Glimmer's pull-based change
  tracking system, and encourages better practices that are easier to optimize.

* Hooks which are used to manipulate elements or the DOM in general can be
  removed in favor of element modifiers, which are discussed in detail in the
  next section.

Based on these considerations, we can reduce these hooks to just a setup and
teardown method: `constructor` and `willDestroy`.

#### `constructor`

The native `constructor` method for the class can be used for initial setup of
the component. This effectively replaces `init`, and allows users to setup state
before _any_ renders occur. It has the following timing semantics:

* **Always**
  * called when a component is created
  * called _before_ any child components are created
  * called _before_ any element modifiers with install hooks in the component's
    template

In many cases, using the `constructor` directly will not be necessary due to
class fields, whose initializers run during instance construction.

```js
class Person {
  constructor() {
    this.name = 'Tomster';
  }
}
```

Is the same as:

```js
class Person {
  name = 'Tomster';
}
```

Class fields are assigned _after_ the call to `super` in the constructor, but
_before_ any of the user's code runs, allowing their values to be accessed by
users as well.

#### `willDestroy`

This hook runs when the component is being destroyed, and can be used for
cleanup code. It has the following timing semantics:

* **Always**
  * called when a component is removed
  * called _after_ any child component `willDestroy` hooks
  * called _after_ any element modifiers with destroy hooks in the component's
    template
  * called _after_ `isDestroying` has been set to `true`, and _before_
    `isDestroyed` has been set to `true`
  * called _after_ the DOM has been fully removed and is inaccessible
* **May or May Not**
  * be called in a stable order relative to sibling component `willDestroy`
    hooks

### Element Modifiers

DOM manipulation is a hard problem for component-oriented frameworks. We spend a
lot of time crafting elegant, functional, template oriented abstractions that
work very well, up until the point where we have to use an imperative native API
like `addEventListener` or `MutationObserver`. This is not a problem unique to
Ember - the recent introduction of the React Hooks API, and the [various flavors
of hooks that exist](https://nikgraf.github.io/react-hooks/), many of which
accomplish the same thing in slightly different ways, suggests that this is a
_fundamentally difficult problem_ no matter how you tackle it.

This is also evidenced by the sheer _number_ of hooks which have been added to
classic Ember components over time to handle various different use cases, and
the fact that there does not appear to be a general consensus on best practices
for using these hooks. In our audit, we observed the following:

1. `didInsertElement` was commonly used for setting up component state which had
   nothing to do with the element and could have been accomplished in `init`.
2. `didRender` was often used for setting up DOM state once on initial render
   only, instead of `didInsertElement`.
3. `didRender` and `didReceiveAttrs` (or `didUpdate` and `didUpdateAttrs`) were
   used interchangeably for setting up and updating DOM state based on incoming
   argument changes, without strong conventions on when to use one or the other,
   or consideration for which ones fire in SSR (`didReceiveAttrs` and
   `didUpdateAttrs`) and which do not.
4. Libraries like
   [ember-lifeline](https://github.com/ember-lifeline/ember-lifeline) were not
   uncommon for managing the extra state that using hooks inevitably creates,
   and imply that it is not always intuitive or well understood that you must
   clean up that state.
5. Guards for SSR appear sporadically throughout various hooks, since some
   (`didInsertElement`, `willDestroyElement`) do _not_ run in SSR, but others
   (`didReceiveAttrs`, `didUpdateAttrs`) do. This adds _another_ layer of state
   that developers must be aware of as they are using lifecycle hooks. Often
   times these guards occured even in hooks which _did not run_ in SSR, implying
   that it is difficult to remember which hooks are best to use in either
   situation.
6. Hooks such as `didRender` had many different potential use cases. It was used
   for reacting to changes to component arguments in some cases, but in others
   it was used as a more general purpose "bloom filter", allowing the component
   to react to _any_ changes to the DOM subtree. The variety of use cases seemed
   to add to the confusion about which hooks should be used in which
   circumstances.
7. Another disadvantage of the flexibility of these hooks was that often
   developers had to add additional validation steps for their specific use
   case. For instance, if a developer wanted to react to a change to a specific
   argument in `didRender` or `didReceiveAttrs`, they had to add cacheing and
   comparison logic manually to do so for each property.

In summary, lifecycle hooks attempted to provide on general solution to the
problem of DOM manipulation for _all_ use-cases, and in doing so provided a
solution that solves each individual problem and use-case in a mediocre way.
Rather than continue these patterns in Glimmer components, we believe that they
should lean instead on _Element Modifiers_.

Modifiers provide a single unified way to define multiple _different_ APIs for
interacting with the DOM. Individual modifiers can be targeted toward specific
use cases, such as adding an event listener or `MutationObserver`, triggering a
callback during certain lifecycle events, capturing element references for use
in components, and more. Importantly, modifiers are _easy to compose_ and
_self-contained_, meaning that it will be possible for general purpose addons to
be built for various use cases, and for them all to be used together without
difficulty.

#### Conversion and Path Forward

Modifiers may be the general purpose solution for writing DOM APIs, but average
Ember developers should not have to _write_ a modifier very often. This is a key
distinction - it means that beginner Ember developers will not need to learn the
ins and outs of modifiers as soon as they need to use DOM, and that they will be
able to instead rely on established patterns from established libraries, similar
to helpers. This combined with the fact that DOM manipulation was on average a
_rare_ occurence in our audits means they won't be overwhelming to learn.

However, while we have merged the [Modifier Manager
RFC](https://github.com/emberjs/rfcs/blob/master/text/0373-Element-Modifier-Managers.md),
the final API for modifiers themselves is still in RFC, and the community hasn't
had a chance to experiment with them and develop patterns yet. We also want to
be able to provide straightforward upgrade and migration guides for users who
want to convert from classic component lifecycle hooks to modifiers. In order to
cover this gap while the community is still absorbing the new APIs, the
modifiers proposed in the [Render Element Modifiers
RFC](https://github.com/emberjs/rfcs/pull/415) will be released as an official
Ember addon. These essentially expose the three hooks of modifiers to users
directly, allowing them to pass callbacks from their components:

```hbs
<div
  {{did-insert this.setupElement @arg1 @arg2}}
  {{did-update this.updateElement @arg1 @arg2}}
  {{will-destroy this.teardownElement}}
>
  ...
</div>
```

These modifiers should allow users to approximate most of the existing lifecycle
hooks, and in most cases should be pretty straightforward to update to. The
Ember guides will provide migration examples for a variety of use cases to
assist in converting to these modifiers. Over time, as addons and libraries are
released that target specific use cases, the guides should be updated to include
popular patterns and demonstrate the most effective and conventional ways to
solve _specific_ problems with DOM manipulation.

### Lifecycle Hook Audit

In the design process of this RFC, we wanted to provide the minimal set of
functionality that covered the previous use cases of classic components. The
final API of Glimmer components as proposed in this RFC is very small, cutting
out almost all existing hooks in favor of a handful of conventional hooks and
element modifiers.

In order to be sure that these hooks and modifiers would cover existing use
cases, we did an audit of a few popular addons: [Ember
Paper](https://miguelcobain.github.io/ember-paper/),
[ember-google-maps](https://github.com/sandydoo/ember-google-maps), and
[liquid-fire](https://github.com/ember-animation/liquid-fire). These libraries
were chosen because they represent a large mix of both common use cases and edge
cases, and give us a decent cross-section of what the hooks are used for today.

We also did a less formal audit of a variety of addons and open source apps,
including [ember-leaflet](https://github.com/miguelcobain/ember-leaflet),
[ember-power-select](https://github.com/cibernox/ember-power-select),
[ember-basic-dropdown](https://github.com/cibernox/ember-basic-dropdown),
[ember-table](https://github.com/Addepar/ember-table),
[vertical-collection](https://github.com/html-next/vertical-collection),
[ember-composablity-tools](https://github.com/miguelcobain/ember-composability-tools),
[Travis Web](https://github.com/travis-ci/travis-web), [the Ghost admin
app](https://github.com/TryGhost/Ghost-Admin), and [Hospital
Run](https://github.com/HospitalRun/hospitalrun-frontend), along with general
code searches through Ember Observer.

In all of these, the only use case we found that was _not_ covered was the
ability to run a hook whenever a render occurs in the subtree of a component
using `didRender` or `didUpdate`. The only instance we found of this was in
[ember-google-maps](https://github.com/sandydoo/ember-google-maps/blob/d901864e9198c1d4956d5ba9629147f64e4ae6b7/addon/templates/components/g-map/overlay.hbs#L6),
where it was used detect when an
[overlay](https://ember-google-maps.sandydoo.me/docs/overlays) component has
rendered and needs to be repositioned. For this rare case, we believe a
[MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)
set to detect mutations to the DOM subtreemay be more appropriate.
Alternatively, a component can be defined with a custom component manager, which
still retains this ability.

The usages from the audit and their equivalent solution in Glimmer components
have been included in this RFC in an appendix.

### Actions

In classic components, actions are defined on the `actions` hash, and can be
referenced in templates using strings passed to the `{{action}}` helper:

```js
export default Component.extend({
  actions: {
    buttonPressed() {
      // ...
    }
  }
})
```
```hbs
<button onclick={{action 'buttonPressed'}}>Press Me!</button>
```

This form of action sending is based on the `ActionHandler` mixin and requires
that the component class have a `send` method. Glimmer components will _not_
implement this API, and as such will not support string based action helpers.
In development mode a special error will be thrown instead, informing users of
alternatives.

Instead, users should use helpers or decorators to bind functions to the
component instance. The `action` helper and modifier do this in templates, as
does the `bind` helper provided by the [ember-bind-helper](https://github.com/Serabe/ember-bind-helper) addon:

```js
export default class ButtonComponent extends Component {
  buttonPressed() {
    // ...
  }
}
```
```hbs
<button onclick={{action this.buttonPressed}}>Press Me!</button>
```

Alternatively, a decorator could be used to bind the helper to the instance,
such as the `@action` decorator proposed in the [Decorators RFC](https://github.com/emberjs/rfcs/pull/408).

```js
export default class ButtonComponent extends Component {
  @action
  buttonPressed() {
    // ...
  }
}
```
```hbs
<button onclick={{this.buttonPressed}}>Press Me!</button>
```

However, one method for binding methods which should be discouraged is assigning
an arrow function to class fields:

```js
export default class ButtonComponent extends Component {
  buttonPressed = () => {
    // ...
  }
}
```

This is messy for a few reasons:

* The method is no longer available on the prototype, making it difficult to
  mock
* It breaks `super` and inheritance, meaning subclasses have no way to override
  the arrow function
* Values such as `arguments` will not be set since it is an arrow function

For more details, see [this document explaining the rationale for decorators](https://github.com/tc39/proposal-decorators/blob/master/bound-decorator-rationale.md)
over class fields for binding.

## Dependencies

In it's current form this RFC is dependent on 2 of 3 open RFCs being accepted:

* The [Decorators RFC](https://github.com/emberjs/rfcs/pull/408) must be
  accepted, because Glimmer components cannot be defined using classic class
  syntax. If it is not accepted this RFC will have to be amended to add a way
  for users to define Glimmer components with classic classes.

* The [Render Element Modifiers RFC](https://github.com/emberjs/rfcs/pull/415)
  must be accepted, since Glimmer components currently do not have any render
  lifecycle hooks or ways to interact with the DOM.

  If it is not accepted, this RFC will have to explore some of the alternatives
  listed below (`{{capture-element}}`, `bounds`, and render hooks).

## How we teach this

Teaching Glimmer components is intrinsically tied to a wider shift in the Ember
programming model - the Ember Octane edition. From a teaching perspective, this
edition will be completely overhauling the guides and updating all of the best
practices as they stand. New users should see Glimmer components as the
_default_, and should not ever have to write a classic component or see one in
the main guides.

Classic components will of course be widely used for some time however, so a
classic section which includes conversion guides and relevant codemods should be
made available in the guides, and maintained for as long as classic components
are supported by Ember.

Breaking down the public API of Glimmer components, we need to cover:

* Native class syntax, including the `constructor` and class fields
* Arguments
* Lifecycle hooks and properties
* Element modifiers

### Native class syntax

One of the benefits of native class syntax is that it is used outside of Ember,
so as time goes on we will be able to assume there is more general knowledge of
it, and provide links to documentation for it for users who are not familiar.
During this transitionary period though we should add a more thorough primer of
the syntax to our guides, and explicitly call out the differences between
classic class syntax and native class syntax, including:

* Usage of `constructor` in classes which _do not_ extend from classic classes.
  Otherwise, always use `init`. This will be tricky, because even when using
  native classes to extend from classic classes, you should still use `init`.

* Nuances of class fields - they run _after_ `super`, and _before_ user code.

* Benefits of class field initializers, and how they can be used to do much of
  the work that would otherwise be done in the `constructor` or `init`

* Expense of class fields - new objects and functions are created for every
  instance, so users should also be careful with them.

* What ends up on the prototype and what's on the instance
* How do property initializers work
* What's the default behavior of a constructor, as it pertains to `super()` and passing arguments
* The risks of anonymous classes and class factories (i.e, you get poor stack traces)
* How to implement "default values" in ES6
* The risks of writing decorators in user-land code (until TC39 stage 4)
* Methods vs arrow function member values
* Getting ahold of prototypes if/when you need them

### Arguments

For arguments, namespacing makes sense in general as an API choice and is common
in other frameworks (`props` in React, etc.). We should be sure to cover this
thoroughly for users who are used to classic components, but it shouldn't
require too much explaining.

Immutability will be a bigger sticking point in general, in particular the
inability to provide default argument values. This is easy enough to work around
using an `{{or}}` helper in templates:

```hbs
Hello, {{or @firstName "friend"}}!
```

Or a defaulting alias getter in the class:

```hbs
Hello, {{this.firstName}}!
```
```js
export default class Greeting extends GlimmerComponent {
  @tracked
  get firstName() {
    return this.args.firstName || 'friend';
  }
}
```

But does add a bit of boilerplate to components. Users will also have to be
careful when attempting to override these "defaults" in subclasses, since it is
not as simple as overriding a class field or property. We can guide users to
use template-only components to "partially applied" components when trying to
provide defaults in subclasses instead, leveraging the outer HTML semantics of
Glimmer components:

```hbs
<!-- components/button.hbs -->
<button class="button {{@type}}">
  <i class="icon {{@type}}"></i>
  {{yield}}
</button>
```
```hbs
<!-- component/success-button.hbs -->
<Button @type="success">{{yield}}</Button>
```
```hbs
<!-- component/danger-button.hbs -->
<Button @type="danger">{{yield}}</Button>
```

Or to explore possibilities using decorators, such as those in the
[sparkles-decorators](https://github.com/gossi/sparkles-decorators) addon.

#### Lifecycle hooks and properties

For users without framework experience, and users of other frameworks, lifecycle
hooks will be very minimal and fairly easy to understand. The lack of render
hooks may be the more difficult part to understand, and we'll have to lean on
the documentation for element modifiers and make sure that is _really_ excellent
to get the concepts there across.

For existing users, who are used to having a variety of hooks to choose from
when coordinating lifecycle events, the hooks may be fairly confusing. The
bullet points here are:

* Tracked properties/computed properties are the primary place to react to
  argument changes for any values that can be computed directly via getters.
  Ideally, most logic for derived state is conventionally in tracked or computed
  properties.
* `willDestroy` is the correct place for all teardown code, like in classic
  components.
* `didReceiveAttrs`, `willRender`, and `didRender` code should be extracted into
  functions which are then passed to `{{did-insert}}` and `{{did-update}}`
* `willDestroyElement` code should be extracted into functions which are then
  passed to `{{will-destroy}}`

Additionally, we should make sure we cover `isDestroying` and `isDestroyed`
pretty thoroughly. Users should know that they can (and probably should) check
these flags if they are doing anything asynchronous that could happen after the
component has been torn down.

#### Element modifiers

The render element modifiers will be the most different part of Glimmer
components for users. The strategy for teaching these is [included in their
RFC](https://github.com/emberjs/rfcs/blob/b9f390cb98f560d9cf876e3b1d67226fe0e1613b/text/0000-render-element-modifiers.md#how-we-teach-this),
but the key points are:

1. Teaching modifiers as a concept first, so users understand that what they're
   looking at is a _general_ tool, and that the render modifiers are an official
   addon provided by Ember.

2. Providing _lots_ of examples for various use cases, especially for users
   transitioning from classic components.

### Template-Only Components

[Template-only components](https://github.com/emberjs/rfcs/blob/master/text/0278-template-only-components.md)
are not strictly speaking related to the `GlimmerComponent` class proposed in
this RFC. However, conceptually they will probably be much easier to teach in
relation to Glimmer components, and will be an important part of Octane that we
should be sure to cover in depth. Additionally, the name of the optional feature
flag, `template-only-glimmer-components`, would make teaching the differences
between Glimmer components and template-only components much more difficult and
confusing.

As such, when writing the documentation for Glimmer components, we should ensure
that we cover template-only components in some detail as well.

## Drawbacks

### Multiple component APIs

One major drawback to Glimmer components is that they add a separate API for
components, meaning that for the forseeable future Ember users will likely
need to learn how to use both interchangeably. This introduces a fair amount of
mental overhead for users, but the benefits of Glimmer components and their
simplicity should make this less problematic.

### Heavy reliance on element modifiers

Glimmer components as proposed in this RFC are heavily reliant on element
modifiers for element manipulation. Element modifiers are a relatively new
concept in Ember, and as such will likely be unfamiliar to users and require
more learning than normal to get used to. This also means that users will not
be able to rely on well established patterns, and will have to develop new ones
for dealing with element manipulation.

### Lack of positional parameter support

Glimmer components are meant to cover _most_ common use cases, but are also
meant to be as minimal as possible. As such, they do _not_ have support for
positional parameters. Positional parameters are already unusable with any
component invoked using angle bracket syntax, but Glimmer components will also
not support them even when using curly bracket invocation.

The use cases for positional parameters are very uncommon, so it doesn't make
sense to add them to the main component class as an option. Instead, we should
make alternative component classes which support positional parameters, perhaps
exclusively (e.g. asserting if positional parameters are _not_ defined).

## Alternatives

### Render lifecycle hooks

The ommission of `didRender`,  `didUpdate`, `didInsertElement`,
`willDestroyElement`, and other render oriented hooks could be confusing to
users. These were staples of classic components, are common in other frameworks,
and make it easy for users to orient themselves when looking at a component
class. They are part of the "standard lifecycle" that make up many component
rendering systems, and make components easier to teach. They also allow users to
place most of their element manipulation logic inside their components, which is
a benefit for users who prefer lighter templates with less logic in them.

Element modifiers, by contrast, are a very new concept in Ember and will require
users to learn a fair amount more just to get started. They force more logic
into the template, and mean users have to look at the template to know if a
method is an element lifecycle hook or an internal method.

Adding the standard element lifecycle hooks would allow users to follow the
patterns they are currently used to, and that are used in other frameworks. If
added without `{{capture-element}}` or `bounds` (see below), they could be used
with `{{did-insert}}` and `{{will-destroy}}` for registering elements:

```hbs
<div {{did-render this.registerElement}}></div>
```
```js
class ExampleComponent extends Component {
  @action
  registerElement(element) {
    this.element = element;
  }

  didRender() {
    setupElement(this.element);
  }
}
```

### Add a `{{capture-element}}` modifier

This alternative would go hand in hand with having render lifecycle hooks.
Rather than relying solely on element modifiers for DOM manipulation, we could
add a modifier that allows users to specify elements which they want to
reference in their component class:

```hbs
<div {{capture-element this}}></div>
```
```js
class ExampleComponent extends Component {
  didRender() {
    setupElement(this.elements.main);
  }
}
```

This would have to take into account multiple usages, and variations of usages.
For instance, how would using `capture-element` in an `if` or `each` work?

```hbs
<div {{capture-element this}}>
  {{#if someBool}}
    <div {{capture-element this 'conditionalElement'}}>
  {{/if}}

  {{#each items as |item|}}
    <div {{capture-element this 'itemElements'}}>
  {{/each}}
</div>
```
```js
class ExampleComponent extends Component {
  didRender() {
    this.elements.main; // the main outer div
    this.elements.conditionalElement; // the conditional element
    this.elements.itemElements; // An array of all the items that are rendered
  }
}
```

This would also mean a fair amount of additional code would need to be added for
reacting to changes in the DOM compared to `{{did-insert}}` and
`{{will-destroy}}`. For instance, if the case of conditionally captured element,
additional validation code will have to exist in `didRender`:

```js
class ExampleComponent extends Component {
  didRender() {
    let { conditionalElement } = this.elements;

    if (conditionalElement) {
      this._previousConditionalElement = conditionalElement;

      setupPlugin(conditionalElement);
    } else {
      teardownPlugin(this._previousConditionalElement);
    }
  }
}
```

This problem is compounded in collections, where any number of elements may be
added or removed.

### Add `element` or `bounds` on the component

We could attempt to add DOM references back to the component, instead of adding
the `{{did-insert}}` and `{{will-destroy}}` modifiers. This would require us to
handle a number of edge-cases (0 element, multi element), and would open up some
intimate details of the Glimmer VM to user code (`bounds` nodes). If in the
future the VM wanted to change these details, it could be problematic.

Element modifiers are less invasive, more declarative, and handle a lot of
boilerplate type code (checking to see if an element exists, for instance).
However, they are also very new to Ember users as a concept (aside from
`{{action}}`) and could be difficult to teach.

### `init` vs `constructor`

Recent changes to the way native classes extend from `EmberObject` made it so
users have to use `init` instead of the `constructor`. This is a pretty
universal caveat currently, so it's fairly teachable - there is a `constructor`,
but use `init` instead (see the [Native Class Constructor
RFC](https://github.com/emberjs/rfcs/blob/master/text/0337-native-class-constructor-update.md))

With the current design of Glimmer components, we are introducing the first base
class which doesn't extend from `EmberObject`, and requires users to use
`constructor` instead. This could be confusing, and will have to be _very_
clearly documented at the least.

We could alternatively include an `init` hook, or have both. This would allow
users to follow one rule for object initialization, but would also lock us into
the supporting the `init` hook for the forseeable future.

### No owner in constructor

Sparkles components do not provide access to the owner or injections in the
constructor, though it is a requested feature. Instead of passing the owner to
the constructor, we could add a `willCreate` or `init` hook which allows users
to setup the instance after the owner has been assigned.

Alternatively, the exact method by which the owner is passed to the constructor
can be changed (on an object vs directly) or all injections could be passed,
enabling typed injections.

## Appendix: Lifecycle Hook Audit

### [ember-paper](https://miguelcobain.github.io/ember-paper/)

#### `didUpdateAttrs`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][ep-did-update-attrs-1]|Setup component state based on incoming arguments|Tracked properties|
|[link][ep-did-update-attrs-2]|Setup component state based on incoming arguments|Tracked properties|
|[link][ep-did-update-attrs-3]|Element setup/update code based on incoming arguments|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-update-attrs-4]|Element setup/update code based on incoming arguments|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-update-attrs-5]|Element setup/update code based on incoming arguments|`{{did-insert}}` and `{{did-update}}`|

[ep-did-update-attrs-1]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-autocomplete-trigger.js#L37
[ep-did-update-attrs-2]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-grid-tile.js#L41
[ep-did-update-attrs-3]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-slider.js#L69
[ep-did-update-attrs-4]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-switch.js#L75
[ep-did-update-attrs-5]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-toast-inner.js#L65

#### `didReceiveAttrs`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][ep-did-receive-attrs-1]|Setup component state based on incoming arguments|Tracked properties and `constructor`|
|[link][ep-did-receive-attrs-2]|Setup component state based on incoming arguments|Tracked properties and `constructor`|
|[link][ep-did-receive-attrs-3]|Setup component state based on incoming arguments, validate incoming arguments|Tracked properties and `constructor`|
|[link][ep-did-receive-attrs-4]|Animate based on incoming arguments|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-receive-attrs-5]|Trigger validations|Tracked properties and `constructor`|
|[link][ep-did-receive-attrs-6]|Element update code and sending an action|Tracked properties and `{{did-insert}}` with args|
|[link][ep-did-receive-attrs-7]|Updating logic and element update code|Tracked properties and `{{did-insert}}` with args|

[ep-did-receive-attrs-1]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-autocomplete.js#L93
[ep-did-receive-attrs-2]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-card-actions.js#L17
[ep-did-receive-attrs-3]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-input.js#L76
[ep-did-receive-attrs-4]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-progress-circular.js#L126
[ep-did-receive-attrs-5]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-select.js#L40
[ep-did-receive-attrs-6]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-sidenav-inner.js#L55
[ep-did-receive-attrs-7]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-virtual-repeat.js#L131

#### `willInsertElement`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][ep-will-insert-element-1]|Container setup code on initialization|`{{did-insert}}`|

[ep-will-insert-element-1]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-toast.js#L91

#### `didInsertElement`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][ep-did-insert-element-1]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-2]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-3]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-4]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-5]|Set focus after initial render|`{{did-insert}}`|
|[link][ep-did-insert-element-6]|Setup animation based on arguments|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-insert-element-7]|Set focus after initial render|`{{did-insert}}`|
|[link][ep-did-insert-element-8]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-9]|Element setup code on initialization (partially based on args)|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-insert-element-10]|Element setup code on initialization|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-insert-element-11]|Element setup code on initialization|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-insert-element-12]|Measure element on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-13]|Element setup code on initialization|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-insert-element-14]|Element setup code on initialization|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-insert-element-15]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-16]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-17]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-18]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-19]|Measure element on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-20]|Element setup code on initialization|`{{did-insert}}`|
|[link][ep-did-insert-element-21]|Element animation on setup|`{{did-insert}}`|

[ep-did-insert-element-1]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-dialog-inner.js#L39
[ep-did-insert-element-2]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-dialog.js#L64
[ep-did-insert-element-3]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-grid-list.js#L52
[ep-did-insert-element-4]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-input.js#L89
[ep-did-insert-element-5]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-menu-content-inner.js#L26
[ep-did-insert-element-6]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-progress-circular.js#L118
[ep-did-insert-element-7]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-select-menu-inner.js#L29
[ep-did-insert-element-8]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-select-options.js#L14
[ep-did-insert-element-9]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-sidenav-inner.js#L48
[ep-did-insert-element-10]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-slider.js#L62
[ep-did-insert-element-11]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-switch.js#L56
[ep-did-insert-element-12]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-tab.js#L44
[ep-did-insert-element-13]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-tabs.js#L71
[ep-did-insert-element-14]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-toast-inner.js#L58
[ep-did-insert-element-15]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-toast.js#L96
[ep-did-insert-element-16]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-tooltip-inner.js#L24
[ep-did-insert-element-17]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-tooltip.js#L67
[ep-did-insert-element-18]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-virtual-repeat-scroller.js#L8
[ep-did-insert-element-19]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-virtual-repeat.js#L8
[ep-did-insert-element-20]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/mixins/ripple-mixin.js#L8
[ep-did-insert-element-21]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/mixins/translate3d-mixin.js#L8

#### `willDestroyElement`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][ep-will-destroy-element-1]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-2]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-3]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-4]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-5]|Teardown animations on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-6]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-7]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-8]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-9]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-10]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-11]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-12]|Element teardown code on destruction|`{{will-destroy}}`|
|[link][ep-will-destroy-element-13]|Unregister child class from parent|`willDestroy`|
|[link][ep-will-destroy-element-14]|Teardown animations on destruction|`{{will-destroy}}`|

[ep-will-destroy-element-1]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-dialog-inner.js#L51
[ep-will-destroy-element-2]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-dialog.js#L79
[ep-will-destroy-element-3]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-grid-list.js#L64
[ep-will-destroy-element-4]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-input.js#L104
[ep-will-destroy-element-5]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-progress-circular.js#L153
[ep-will-destroy-element-6]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-slider.js#L81
[ep-will-destroy-element-7]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-switch.js#L70
[ep-will-destroy-element-8]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-tabs.js#L102
[ep-will-destroy-element-9]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-toast-inner.js#L77
[ep-will-destroy-element-10]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-toast.js#L118
[ep-will-destroy-element-11]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-tooltip.js#L110
[ep-will-destroy-element-12]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-virtual-repeat-scroller.js#L17
[ep-will-destroy-element-13]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/mixins/child-mixin.js#L30
[ep-will-destroy-element-14]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/mixins/translate3d-mixin.js#L30

#### `didUpdate`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][ep-did-update-1]|Reapply styles based on changes to args|`{{did-insert}}` and `{{did-update}}`|

[ep-did-update-1]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-grid-list.js#L57

#### `didRender`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][ep-did-render-1]|Resize component based on changes to args|`{{did-insert}}` and `{{did-update}}`, or [MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)|
|[link][ep-did-render-2]|Animate component based on changes to arguments|`{{did-insert}}` and `{{did-update}}`|
|[link][ep-did-render-3]|Set `elementDidRender` boolean on instance|`{{did-insert}}`|
|[link][ep-did-render-4]|Measure component on render|`{{did-insert}}` and `{{did-update}}`, or [MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)|
|[link][ep-did-render-5]|Resize component based on changes to size|`{{did-insert}}` with args, or [MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver)|
|[link][ep-did-render-6]|Measure element sizes based on changes to args|`{{did-insert}}` with args|

[ep-did-render-1]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-input.js#L97
[ep-did-render-2]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-speed-dial-action-action.js#L34
[ep-did-render-3]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-speed-dial.js#L32
[ep-did-render-4]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-tab.js#L52
[ep-did-render-5]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-tabs.js#L96
[ep-did-render-6]: https://github.com/miguelcobain/ember-paper/blob/b36f7a7b5e19c60fa71e945590178d86274bd49d/addon/components/paper-virtual-repeat.js#L156

### [ember-google-maps](https://github.com/sandydoo/ember-google-maps)

#### `didUpdateAttrs`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][egm-did-update-attrs-1]|Synchronize options with Google maps|Refactor to use actions to modify data, or use a modifier|
|[link][egm-did-update-attrs-2]|Update component based on changes to arguments|Refactor to use actions to modify data, or use a modifier|

[egm-did-update-attrs-1]: https://github.com/sandydoo/ember-google-maps/blob/3f846751eda7fff08a02367c04407cf545741f00/addon/components/g-map.js#L103
[egm-did-update-attrs-2]: https://github.com/sandydoo/ember-google-maps/blob/3f846751eda7fff08a02367c04407cf545741f00/addon/components/g-map/map-component.js#L54

#### `didReceiveAttrs`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][egm-did-receive-attrs-1]|Register component with parent|`constructor`|

[egm-did-receive-attrs-1]: https://github.com/sandydoo/ember-google-maps/blob/3f846751eda7fff08a02367c04407cf545741f00/addon/components/g-map/waypoint.js#L20

#### `didInsertElement`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][egm-did-insert-element-1]|Register component with parent and initialize|`constructor` and parent component `{{did-insert}}`|

[egm-did-insert-element-1]: https://github.com/sandydoo/ember-google-maps/blob/3f846751eda7fff08a02367c04407cf545741f00/addon/components/g-map/map-component.js#L46

#### `willDestroyElement`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][egm-will-destroy-element-1]|Element teardown code on destruction (and potentially destruction of parent)|`willDestroy` and parent component `{{will-destroy}}`|
|[link][egm-will-destroy-element-2]|Unregister element from parent|`willDestroy` and parent component `{{will-destroy}}`|
|[link][egm-will-destroy-element-3]|Teardown class state|`willDestroy`|
|[link][egm-will-destroy-element-4]|Teardown class state|`willDestroy`|

[egm-will-destroy-element-1]: https://github.com/sandydoo/ember-google-maps/blob/3f846751eda7fff08a02367c04407cf545741f00/addon/components/g-map/map-component.js#L60
[egm-will-destroy-element-2]: https://github.com/sandydoo/ember-google-maps/blob/3f846751eda7fff08a02367c04407cf545741f00/addon/components/g-map/waypoint.js#L26
[egm-will-destroy-element-3]: https://github.com/sandydoo/ember-google-maps/blob/3f846751eda7fff08a02367c04407cf545741f00/addon/mixins/process-options.js#L26
[egm-will-destroy-element-4]: https://github.com/sandydoo/ember-google-maps/blob/3f846751eda7fff08a02367c04407cf545741f00/addon/mixins/register-events.js#L26

#### `didRender`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][egm-did-render-1]|Detect changes to subtree and reposition overlay|[MutationObserver](https://developer.mozilla.org/en-US/docs/Web/API/MutationObserver) or custom component that can trigger actions on subtree rerenders|

[egm-did-render-1]: https://github.com/sandydoo/ember-google-maps/blob/master/addon/templates/components/g-map/overlay.hbs#L6

### [liquid-fire](https://github.com/ember-animation/liquid-fire)

#### `didReceiveAttrs`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][lf-did-receive-attrs-1]|Capture argument as component state|`constructor`|
|[link][lf-did-receive-attrs-2]|Capture argument as component state|`constructor`|
|[link][lf-did-receive-attrs-3]|Run update code for changing versions (and animating)|`constructor` and tracked properties or element modifiers|

[lf-did-receive-attrs-1]: https://github.com/ember-animation/liquid-fire/blob/21711359faf0a396489f169ae34c1637e7f45828/addon/components/illiquid-model.js#L7
[lf-did-receive-attrs-2]: https://github.com/ember-animation/liquid-fire/blob/21711359faf0a396489f169ae34c1637e7f45828/addon/components/liquid-outlet.js#L23
[lf-did-receive-attrs-3]: https://github.com/ember-animation/liquid-fire/blob/21711359faf0a396489f169ae34c1637e7f45828/addon/components/liquid-versions.js#L15

#### `didInsertElement`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][lf-did-insert-element-1]|Trigger animation|`{{did-insert}}`|
|[link][lf-did-insert-element-2]|Set did render|`{{did-insert}}`|
|[link][lf-did-insert-element-3]|Element setup code on insertion|`{{did-insert}}`|
|[link][lf-did-insert-element-4]|Element setup code on insertion|`{{did-insert}}`|
|[link][lf-did-insert-element-5]|Pause animations on insertion (continue later via action)|`{{did-insert}}`|

[lf-did-insert-element-1]: https://github.com/ember-animation/liquid-fire/blob/21711359faf0a396489f169ae34c1637e7f45828/addon/components/liquid-child.js#L12
[lf-did-insert-element-2]: https://github.com/ember-animation/liquid-fire/blob/21711359faf0a396489f169ae34c1637e7f45828/addon/components/liquid-container.js#L44
[lf-did-insert-element-3]: https://github.com/ember-animation/liquid-fire/blob/21711359faf0a396489f169ae34c1637e7f45828/addon/components/liquid-measured.js#L15
[lf-did-insert-element-4]: https://github.com/ember-animation/liquid-fire/blob/21711359faf0a396489f169ae34c1637e7f45828/addon/components/liquid-spacer.js#L11
[lf-did-insert-element-5]: https://github.com/ember-animation/liquid-fire/blob/21711359faf0a396489f169ae34c1637e7f45828/addon/components/liquid-sync.js#L8

#### `willDestroyElement`

|Usage|Use Case|Converts To|
|-----|--------|-----------|
|[link][lf-will-destroy-element-1]|Element teardown code on destruction|`{{will-destroy}}`|

[lf-will-destroy-element-1]: https://github.com/ember-animation/liquid-fire/blob/21711359faf0a396489f169ae34c1637e7f45828/addon/components/liquid-measured.js#L37
