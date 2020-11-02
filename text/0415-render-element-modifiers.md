---
Start Date: 2018-12-13
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/415
Tracking: https://github.com/emberjs/rfc-tracking/issues/8

---

# Render Element Modifiers

## Summary

Element modifiers are a recently introduced concept in Ember that allow users to
run code that is tied to the lifecycle of an _element_ in a template, rather
than the component's lifecycle. They allow users to write self-contained logic
for manipulating the state of elements, and in many cases can be fully
independent of component code and state.

However, there are many cases where users wish to run some component code when
an element is setting up or tearing down. Today, this logic conventionally lives
in the `didInsertElement`, `didRender`, `didUpdate`, and  `willDestroyElement`
hooks in components, but there are cases where these hooks are not ideal.

This RFC proposes creating an official Ember addon which provides three new
generic element modifiers: `{{did-insert}}`, `{{did-update}}`, and
`{{will-destroy}}`. Users will be able to use these to run code during the most
common phases of any element's lifecycle.

## Motivation

The primary component hooks for interacting with the DOM today are:

* `didInsertElement`
* `didRender`
* `didUpdate`
* `willDestroyElement`

These render hooks cover many use cases. However, there are some cases which
they do not cover, such as setting up logic for conditional elements, or tagless
components. There also is no easy way to share element setup logic, aside from
mixins or pure functions (which require some amount of boilerplate).

### Conditionals

Render code for elements which exist conditionally is fairly tricky. Consider a
simple popover component:

```hbs
{{#if this.isOpen}}
  <div class="popover">
    {{yield}}
  </div>
{{/if}}
```

If the developer decides to use an external library like [Popper.js](https://popper.js.org)
to position the popover, they have to add a fair amount of boilerplate. On each
render, they need to check if the popover was added to the DOM or removed from
it, and setup or teardown accordingly.

```js
export default Component.extend({
  didRender() {
    if (this.isOpen && !this._popper) {
      let popoverElement = this.element.querySelector('.popover');

      this._popper = new Popper(document, popoverElement);
    } else if (this._popper) {
      this._popper.destroy();
    }
  },

  willDestroyElement() {
    if (this._popper) {
      this._popper.destroy();
    }
  }
});
```

At this level of complexity, most developers would reasonably choose to create
a second component to be used within the `{{if}}` block so they can use standard
lifecycle hooks. Sometimes this makes sense as it helps to separate concerns and
organize code, but other times it is clearly working around the limitations of
render hooks, and can feel like more components are being created than are
necessary.

With render modifiers, hooks are run whenever the _element_ they are applied to
is setup and torn down, which means we can focus on the setup and teardown code
without worrying about the overall lifecycle:

```hbs
{{#if this.isOpen}}
  <div
    {{did-insert (action this.setupPopper)}}
    {{will-destroy (action this.teardownPopper)}}

    class="popover"
  >
    {{yield}}
  </div>
{{/if}}
```
```js
export default Component.extend({
  setupPopper(element) {
    this._popper = new Popper(document, element);
  },

  teardownPopper() {
    this._popper.destroy();
  }
});
```

The element that the modifiers are applied to is also passed to the function, so
there is no longer a need to use `querySelector`. Overall the end result is a
fair amount simpler, without the need for an additional component.

These same issues are also present for collections items within an `{{each}}`
loop, and the render modifiers can be used to solve them as well:

```hbs
<ul>
  {{#each items as |item|}}
    <li
      {{did-insert (action this.registerElement)}}
      {{will-destroy (action this.unregisterElement)}}
    >
      ...
    </li>
  {{/each}}
</ul>
```

### Tagless Components

Additionally, render hooks do not provide great support for tagless components
(`tagName: ''`). While the hooks fire when the component is rendered, they have
no way to target any of the elements which are in the component's template,
meaning users must use `querySelector` and setup some unique id or class to
target the element by:

```js
export default Component.extend({
  tagName: '',

  listId: computed(function() {
    return generateId();
  }),

  didInsertElement() {
    let element = document.querySelector(`#${this.listId}`);

    // ...
  },

  willDestroyElement() {
    let element = document.querySelector(`#${this.listId}`);

// ...
  }
});
```
```hbs
<ul id={{listId}}>
  ...
</ul>

<div>
  ...
</div>
```

The render modifiers can be used to add hooks to the appropriate main element in
tagless components:

```js
export default Component.extend({
  tagName: '',

  didInsertList(element) {
    // ...
  },

  willDestroyList(element) {
    // ...
  }
});
```
```hbs
<ul
  {{did-insert (action this.didInsertList)}}
  {{will-destroy (action this.willDestroyList)}}
>
  ...
</ul>

<div>
  ...
</div>
```

### Reusable Helpers

Currently, the best ways to share element setup code are either via mixins,
which are somewhat opaque and can encourage problematic patterns, or standard JS
functions, which generally require some amount of boilerplate.

Developers will be able to define element modifiers in the future with modifier
managers provided by addons. However, the proposed modifier APIs are fairly
verbose (with good reason) and not stabilized.

However, the render modifiers can receive _any_ function as their first
parameter, allowing users to share and reuse common element setup code with
helpers. For instance, a simple `scrollTo` helper could be created to set the
scroll position of an element:

```js
// helpers/scroll-to.js
export default function scrollTo() {
  return (element, [scrollPosition]) => element.scrollTop = scrollPosition;
}
```
```hbs
<div
  {{did-insert (scroll-to) @scrollPosition}}
  {{did-update (scroll-to) @scrollPosition}}
  class="scroll-container"
>
  ...
</div>
```

### Official Addon

While these modifiers will be generally useful, modifiers are meant to be a more
generic API that can be used to create libraries for solving specific problems.
Unfortunately, the community hasn't had much time to experiment with modifiers,
since the public API for them hasn't been finalized.

The modifiers in this RFC will provide an basic stepping stone for users who
want to emulate lifecycle hooks and incrementally convert their applications to
modifiers while modifiers in general are being experimented with in the
community. In time, users should be able to pick and choose the modifiers that
suit their needs more directly and effectively, and they shouldn't have to
include these modifiers in the payload. These modifiers should also not be seen
as the "Ember way" - they are just another addon, a basic one supported by
the Ember core team, but one which may or may not be appropriate for a given
application.

## Detailed design

This RFC proposes adding three element modifiers:

* `{{did-insert}}`
* `{{did-update}}`
* `{{will-destroy}}`

Note that element modifiers do _not_ run in SSR mode - this code is only run on
clients. Each of these modifiers receives a callback as it's first positional
parameter:

```ts
type RenderModifierCallback = (element: Element, positionalArgs: [any], namedArgs: object): void;
```

The `element` argument is the element that the modifier is applied to,
`positionalArgs` contains any remaining positional arguments passed to the
modifier besides the callback, and `namedArgs` contains any named arguments
passed to the modifier. If the first positional argument is not a callable
function, the modifier will throw an error.

> Note: The timing semantics in the following section were mostly defined in the
> [element modifier manager RFC](https://github.com/emberjs/rfcs/blob/master/text/0373-Element-Modifier-Managers.md)
> and are repeated here for clarity and convenience.

### `{{did-insert}}`

This modifier is activated only when The element is inserted in the DOM.

It has the following timing semantics when activated:

* **Always**
  * called after DOM insertion
  * called _after_ any child element's `{{did-insert}}` modifiers
  * called _after_ the enclosing component's `willRender` hook
  * called _before_ the enclosing component's `didRender` hook
  * called in definition order in the template
* **May or May Not**
  * be called in the same tick as DOM insertion
  * have the sibling nodes fully initialized in DOM

Note that these statements do not refer to when the modifier is _activated_,
only to when it will be run relative to other hooks and modifiers _should it be
activated_. The modifier is only activated on insertion.

### `{{did-update}}`

This modifier is activated only on _updates_ to it's arguments (both positional
and named). It does _not_ run during or after initial render, or before
element destruction.

It has the following timing semantics when activated:

* **Always**
  * called after the arguments to the modifier have changed
  * called _after_ any child element's `{{did-update}}` modifiers
  * called _after_ the enclosing component's `willUpdate` hook
  * called _before_ the enclosing component's `didUpdate` hook
  * called in definition order in the template
* **Never**
  * called if the arguments to the modifier are constants

### `{{will-destroy}}`

This modifier is activated:

1. immediately before the element is removed from the DOM.

It has the following timing semantics when activated:

* **Always**
  * called _after_ any child element's `{{will-destroy}}` modifiers
  * called _before_ the enclosing component's `willDestroy` hook
  * called in definition order in the template
* **May or May Not**
  * be called in the same tick as DOM removal

### Function Binding

Functions which are passed to these element modifiers will _not_ be bound to any
context by default. Users can bind them using the `(action)` helper:

```hbs
<div {{did-insert (action this.setupElement)}}></div>
```

Or by using the `@action` decorator provided by the
[Decorators RFC](https://github.com/emberjs/rfcs/pull/408) to bind the function
in the class itself:

```js
export default class ExampleComponent extends Component {
  @action
  setupElement() {
    // ...
  }
}
```
```hbs
<div {{did-insert this.setupElement}}></div>
```

## How we teach this

Element modifiers will be new to everyone, so we're starting with a mostly blank
slate. The only modifier that exists in classic Ember is `{{action}}`, and while
most existing users will be familiar with it, that familiarity may not translate
to the more general idea of modifiers.

The first thing we should focus on is teaching _modifiers in general_. Modifiers
should be seen as the place for any logic which needs to act directly on an
element, or when an element is added to or removed from the DOM. Modifiers can
be fully independent (for instance, a `scroll-to` modifier that transparently
manages the scroll position of the element) or they can interact with the
component (like the `did-insert` and `will-destroy` modifiers). In all cases
though, they are _tied to the render lifecycle of the element_, and they
generally contain _side-effects_ (though these may be transparent and
declarative, as in the case of `{{action}}` or the theoretical `{{scroll-to}}`).

Second, we should teach the render modifiers specifically. We can do this by
illustrating common use cases which can currently be solved with render hooks,
and comparing them to using modifiers for the same solution. We should also
emphasize that these are an addon, not part of the core framework, and are
useful as solutions for _specific_ problems. As more modifiers become available,
we should create additional guides that focus on using the _best_ modifier for
the job, rather than these generic ones.

One thing we should definitely avoid teaching except in advanced cases is the
_ordering_ of element modifiers. Ideally, element modifiers should be
commutative, and order should not be something users have to think about. When
custom element modifiers become widely available, this should be considered best
practice.

### Example: Scrolling an element to a position

This sets the scroll position of an element, and updates it whenever the scroll
position changes.

Before:

```hbs
{{yield}}
```
```js
export default Component.extend({
  classNames: ['scroll-container'],

  didRender() {
    this.element.scrollTop = this.scrollPosition;
  }
});
```

After:

```hbs
<div
  {{did-insert this.setScrollPosition @scrollPosition}}
  {{did-update this.setScrollPosition @scrollPosition}}

  class="scroll-container"
>
  {{yield}}
</div>
```
```js
export default class Component.extend({
  setScrollPosition(element, scrollPosition) {
    element.scrollTop = scrollPosition;
  }
})
```

#### Example: Adding a class to an element after render for CSS animations

This adds a CSS class to an alert element in a conditional whenever it renders
to fade it in, which is a bit of an extra hoop. For CSS transitions to work, we
need to append the element _without_ the class, then add the class after it has
been appended.

Before:

```hbs
{{#if shouldShow}}
  <div class="alert">
    {{yield}}
  </div>
{{/if}}
```
```js
export default Component.extend({
  didRender() {
    let alert = this.element.querySelector('.alert');

    if (alert) {
      alert.classList.add('fade-in');
    }
  }
});
```

After:

```hbs
{{#if shouldShow}}
  <div {{did-insert this.fadeIn}} class="alert">
    {{yield}}
  </div>
{{/if}}
```
```js
export default Component.extend({
  fadeIn(element) {
    element.classList.add('fade-in');
  }
});
```

#### Example: Resizing text area

One key thing to know about `{{did-update}}` is it will not rerun whenever the
_contents_ or _attributes_ on the element change. For instance, `{{did-update}}`
will _not_ rerun when `@type` changes here:

```hbs
<div {{did-update this.setupType}} class="{{@type}}"></div>
```

If `{{did-update}}` should rerun whenever a value changes, the value should be
passed as a parameter to the modifier. For instance, a textarea which wants to
resize itself to fit text whenever the text is modified could be setup like
this:

```hbs
<textarea {{did-update this.resizeArea @text}}>
  {{@text}}
</textarea>
```
```js
export default Component.extend({
  resizeArea(element) {
    element.css.height = `${element.scrollHeight}px`;
  }
});
```

#### Example: `ember-composability-tools` style rendering

This is the type of rendering done by libraries like `ember-leaflet`, which use
components to control the _rendering_ of the library, but without any templates
themselves. The underlying library for this is [here](https://github.com/miguelcobain/ember-composability-tools).
This is a simplified example of how you could accomplish this with Glimmer
components and element modifiers.

Node component:

```js
// components/node.js
export default Component.extend({
  init() {
    super(...arguments);
    this.children = new Set();

    this.parent.registerChild(this);
  }

  willDestroy() {
    super(...arguments);

    this.parent.unregisterChild(this);
  }

  registerChild(child) {
    this.children.add(child);
  }

  unregisterChild(child) {
    this.children.delete(child);
  }

  didInsertNode(element) {
    // library setup code goes here

    this.children.forEach(c => c.didInsertNode(element));
  }

  willDestroyNode(element) {
    // library teardown code goes here

    this.children.forEach(c => c.willDestroyNode(element));
  }
}
```
```hbs
<!-- components/node.hbs -->
{{yield (component "node" parent=this)}}
```

Root component:

```js
// components/root.js
import NodeComponent from './node.js';

export default NodeComponent.extend();
```
```hbs
<!-- components/root.hbs -->
<div
  {{did-insert (action this.didInsertNode)}}
  {{will-destroy (action this.willDestroyNode)}}
>
  {{yield (component "node" parent=this)}}
</div>
```

Usage:

```hbs
<Root as |node|>
  <node as |node|>
    <node />
  </node>
</Root>
```

## Drawbacks

* Adding these modifiers means that there are more ways to accomplish similar
  goals, which may be confusing to developers. It may be less clear which is the
  conventional solution in a given situation.

* Relying on users binding via `action` is somewhat unintuitive, and may feel
  like it's getting in the way, especially considering sometimes methods will
  work without binding (if they never access `this`).

## Alternatives

* Stick with only lifecycle hooks for these situations, and don't add generic
  modifiers for them.


