- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Element Modifiers

## Summary

This RFC introduces the concept of user defined element modifiers. Unlike a component, there is no template/layout for an element modifier. Unlike a helper, an element modifier does not return a value.

Below is an example of the element modifier syntax:

```hbs
<button {{add-event-listener 'click' (action 'save')}}>Save</button>
```

This RFC supercedes the [original element modifiers RFC](https://github.com/emberjs/rfcs/pull/112) and is intended to replace the [`this.bounds` RFC](https://github.com/emberjs/rfcs/pull/351).

## Motivation

`Ember.Component` instances have a `this.element` property which provides you a single DOM node as defined by `tagName`. The children of this node will be the DOM representation of what you wrote in your template. Templates are typically referred to having `innerHTML` semantics in `Ember.Component`s since there is a single wrapping element that is the parent of the template. These semantics allow for components to encapsulate some 3rd party JavaScript library or do some fine grain DOM manipulation.

Custom components have `outerHTML` semantics, meaning what you see in the template is what you get in the DOM, there is no `tagName` that wraps the template. While this drastically simplifies the API for creating new components it makes reliable access to the a component's DOM structure very difficult to do. As [pointed out](https://github.com/emberjs/rfcs/pull/351#issuecomment-412123046) in the [Bounds RFC](https://github.com/emberjs/rfcs/pull/351) the stability of the nodes in the `Bounds` object creates way too many footguns. So a formalized way for accessing DOM in the custom component world is still needed.

Element modifiers allow for stable access of the DOM node they are installed on. This allows for programatic access to DOM in custom component templates and also offers a more targeted construct for cases where classic components were being used. The introduction of this API will likely result in the proliferation of one or several popular addons for managing element event listeners, style and animation.

## Detailed design

### Invocation

An element modifier is invoked in "element space". This is the space between `<` and `>` opening an HTML tag. For example:

```hbs
<button {{flummux}}></button>
<span {{whipperwill 'carrot'}}><i>Some DOM</i></span>
<b {{crum bing='whoop'}} zip="bango">Hm...</b>
```

Element modifiers may be invoked with positional and/or named arguments.

### Definition and lookup

A basic element modifier is defined with the type of `modifier`. For example these paths would be global element modifiers in an application:

```
Classic paths:

  app/modifiers/flummux.js
  app/modifiers/whipperwill.js

MU paths:

  src/ui/components/flummux/modifier.js
  src/ui/routes/posts/-components/whipperwill/modifier.js
```

In Module Unification, modifiers live within the generalized collection type "components" [as specified](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md#components). Modifiers, like component and helpers, are eligible for local lookup. For example:

```
MU paths:

  src/ui/routes/posts/-components/post-editor/flummux/modifier.js
```

The element modifier class is a default export from these files. For example:

```js
import Modifier from '@ember/modifier';

export default class extends Modifier {}
```

### Lifecycle Hooks

During rendering and teardown of a target element, any attached element modifiers will execute a series of hooks. These hooks are:

* `didInsertElement`
* `didUpdate`
* `willDestroyElement`

It is important to note that in server-side rendering environments none of the lifecycle events are called.

#### `didInsertElement` semantics

`didInsertElement` is called when the element is in the DOM and it's children are attached. The element is available on `this.element` of the modifier instance. Unlike classic components, the `didInsertElement` hook receives the positional and named params as arguments in the same way helpers do. This hook is only called once.

This hook has the following timing semantics:

**Always**
- called **after** all children modifiers `didInsertElement` hook are called
- called **after** DOM insertion

 **May or May Not**
- be called in the same tick as DOM insertion
- have the sibling nodes fully initialized in DOM

Below is an example of how this hook could be used:

```js
import Modifier from '@ember/modifier';

export default class extends Modifier {
  this.listenerOptions = undefined;
  didInsertElement([ eventType, callback ], eventOptions) {
    this.listenerOptions = [
      eventType,
      callBack,
      eventOptions
    ];
    this.element.addEventListener(eventType, callback, eventOptions);
  }
}
```

#### `didUpdate` semantics

`didUpdate` is called whenever any of the parameters used by the modifier are updated. `didUpdate` has the same signature as `didInsertElement`.

This hook has the following timing semantics:

**Always**
- called **after** the arguments to the modifier have changed

**Never**
- called if the arguments to the modifier are constants

Below is an example of how this hook could be used:

```js
import Modifier from '@ember/modifier';

export default class extends Modifier {
  this.listenerOptions = undefined;
  didInsertElement([ eventType, callback ], eventOptions) {
    this.listenerOptions = [
      eventType,
      callBack,
      eventOptions
    ];
    this.element.addEventListener(eventType, callback, eventOptions);
  },

  didUpdate([ eventType, callback ], eventOptions) {
    this.element.removeEventListener(...this.listenerOptions);
    this.element.addEventListener(eventType, callback, eventOptions);
    this.listenerOptions = [eventType, callback, eventOptions];
  }
}
```

#### `willDestroyElement` semantics

`willDestroyElement` is called during the destruction of the element. It receives no arguments.

Below is an example of how this hook could be used:

```js
import Modifier from '@ember/modifier';

export default class extends Modifier {
  this.listenerOptions = undefined;
  didInsertElement([ eventType, callback ], eventOptions) {
    this.listenerOptions = [
      eventType,
      callBack,
      eventOptions
    ];
    this.element.addEventListener(eventType, callback, eventOptions);
  },

  didUpdate([ eventType, callback ], eventOptions) {
    this.element.removeEventListener(...this.listenerOptions);
    this.element.addEventListener(eventType, callback, eventOptions);
    this.listenerOptions = [eventType, callback, eventOptions];
  }

  willDestroyElement() {
    this.element.removeEventListener(...this.listenerOptions);
  }
}
```

This hook has the following timing semantics:

**Always**
- called **after** all children modifier's `willDestroyElement` hook is called
- called **after** all children component's `willDestroyComponent` hook is called.

 **May or May Not**
- be called in the same tick as DOM removal

### Relationship With Components

Element modifers are intended to be directly installed onto HTML elements and can not be invoked on a component invocation. They also cannot be passed as arguments to a component. The examples below will result in a compile error.

```hbs
<FooBar {{draggable x=x y=y}} />
```

```hbs
<FooBar (draggable x=x y=y) />
```

This also means that they have no relationship to `...attributes`.

## Examples

Below are some examples of what you could do with a modifier:

### Performance Marking
```hbs
<section id="about-us" {{performance 'mark' 'about-page'}}>
  <h1>About Us</h1>
  {{!-- snip --}}
</section>
```

```js
import Modifier from '@ember/modifier';

export Modifier.extend({
  didInsertElement([ type, marker ]) {
    performance[type](marker);
  }
});
```

### jQuery Widget
```hbs
<input type="date" {{datepicker changeMonth=true changeYear=true}} />
```

```js
import Modifier from '@ember/modifier';
import $ from 'jquery';

export Modifier.extend({
  didInsertElement(_params, hash) {
    $(this.element).datepicker(this._normalizeOptions(hash));
  },

  _normalizeOptions(options) {
    return Object.assign(options, { minDate: 20, maxDate: '+1M +10D' });
  },

  willDestroyElement() {
    $(this.element).datepicker('destroy');
  }
});
```

### Page View Tracking
```hbs
<section {{track-impression eventCategory="Post"}}>
  <header>Chad liked a post</header>
  <img src="cat.jpg">
  {{!-- Snip --}}
<section>
```

```js
import Modifier from '@ember/modifier';
import { inject as service } from '@ember/service';

export Modifier.extend({
  ga: service('google-analytics'),
  init() {
    this._super(...arguments);
    this.eventCategory = undefined;
    this.interSectionObserver = new IntersectionObserver(entries => {
      entries.forEach(entry => {
        this.ga.send('event', 'impression', this.eventCategory);
      });
    });
  },
  didInsertElement(_params, { eventCategory }) {
    this.eventCategory = eventCategory;
    this.interSectionObserver.observe(this.element);
  },

  willDestroyElement() {
    this.interSectionObserver.unobserve(this.element);
  }
});
```

## How we teach this

While user defined element modifiers are a new concept, Ember has had the [`{{action}}` modifier](https://www.emberjs.com/api/ember/release/classes/Ember.Templates.helpers/methods/action?anchor=action) since 1.0.0. We would need to update the existing documentation around `{{action}}` to refer to it as a modifier. In terms of documenting the modifier base class, I think we can largely repurpose existing documentation around [component's `didInsertElement`](https://guides.emberjs.com/release/components/the-component-lifecycle/#toc_integrating-with-third-party-libraries-with-didinsertelement) hook.

In terms of guides, I believe we should add a section to the "Templates" section to outline how to write modifiers. This would be similar to the ["Writing Helpers"](https://guides.emberjs.com/release/templates/writing-helpers/) guide.

## Drawbacks

The drawbacks of adding element modifiers largely deal with explaining when to use a classic component for encapsulating some DOM manipulation versus when to use an element modifier. That being said it is likely that the recommendation would be to use element modifiers if you are just manually modifying the DOM.

This API also doesn't attempt to create a corollary of `this.element` for custom components and instead offers a different API for altering DOM nodes directly. This expands the surface area of Ember's API.

## Alternatives

The alternative to this is to create a "ref"-like API that is available in [other client side frameworks](https://reactjs.org/docs/refs-and-the-dom.html). This may look like the following:

```hbs
<section>
  <h1 {{ref "heading"}}>Hello!</h1>
  <p>How are you?</p>
</section>
```

```js
import Component from '@glimmer/component';

export class extends Component {
  headingNode = null;
  heading(headingElement) {
    if (headingElement) {
      this.headingElement = headingElement;
      headingElement.addEventListener('click', this.click);
    } else {
      this.headingElement.removeEventListener('click', this.click);
      this.headingElement = null;
    }
  }

  click(evt) {
    evt.preventDefault();
    alert('click');
  }
}
```

This API would be implemented as a framework level modifier and would call hooks on the backing class with the element the `{{ref}}` was installed on. When the element is being removed the method would be called with `null`. It's the component author's responsibility to manage the DOM node.

This RFC does not close the door on a `{{ref}}`-like API, but rather exposes the primitives needed to create it.

## Unresolved questions

TBD?
