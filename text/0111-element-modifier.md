- Start Date: 2016-01-23
- RFC PR: https://github.com/emberjs/rfcs/pull/111
- Ember Issue:

# Summary

As an outcome of [RFC 100](https://github.com/emberjs/rfcs/pull/100#issuecomment-172427565)
it was decided that element-space helpers should be added to Ember as a public
API.

In this example, `add-event-listener` fits the syntax of an element modifier:

```hbs
<span {{add-event-listener 'click' (action 'save')}}>Save</span>
```

The implementation of `add-event-listener` would have several hooks called during
rendering, similar to the rendering hooks fired on a component. Unlike a
component, there is no template/layout for an element modifier. Unlike a
helper, an element modifier does not return a value.

# Motivation

Element modifiers allow for user-space implementation of event listeners,
style, and animation tooling that does not exist in Ember today. It brings
back some functionality removed from the framework in Ember 2.0 (when
various intimate helper APIs were removed).

The introduction of this API will likely result in the proliferation of
one or several popular addons for managing element event listeners, style
and animation.

# Detailed design

### Invocation

An element modifier is invoked in "element space". This is the space between
`<` and `>` opening an HTML tag. For example:

```hbs
<button {{flummux}}></button>
<span {{whipperwill 'carrot'}}><i>Some DOM</i></span>
<b {{crum bing='whoop'}} zip="bango">Hm...</b>
```

Element modifiers may be invoked with params or hash arguments.

### Definition and lookup

A basic element modifier is defined with the type of `element-modifier`. For
example these paths would be global element modifiers in an application:

```
Classic paths:

  app/element-modifiers/flummux.js
  app/element-modifiers/whipperwill.js

Pods paths:

  app/flummux/element-modifier.js
  app/whipperwill/element-modifier.js
```

Element modifiers, like component and helpers, are eligible for local lookup.
For example:

```
Pods paths:

  app/posts/index/element-modifiers/flummux.js
```

The element modifier class is a default export from these files. For example:

```js
import Ember from 'ember';

export default Ember.ElementModifier.extend({});
```

### Hooks

During rendering and teardown of a target element, any attached element
modifiers will execute a series of hooks. These hooks are:

* `willInsertElement` (only upon initial render)
* `didUpdateAttrs` (only upon subsequent render)
* `willRender`
* `didRender`
* `didInsertElement` (only upon initial render)
* `willDestroyElement` (only upon teardown)

These lifecycle hooks are similar to those of the component lifecycle. Similarly
the properties of `this.element` and `this.attrs` may be present during the
execution of a hook.

* during `willInsertElement`, initial `this.attrs` will be present, but no `this.element`
* during `didUpdateAttrs` both `this.attrs` and `this.element` are present. Additionally
  the hook is pass `oldAttrs, newAttrs` as arguments.
* during `willRender` both `this.attrs` and `this.element` are present
* during `didRender` both `this.attrs` and `this.element` are present
* during `didInsertElement` both `this.attrs` and `this.element` are present
* during `willDestroyElement` both `this.attrs` and `this.element` are present

An example of a simple element modifier definition:

```hbs
<button {{noodle logEvent='mouseover'}}></button>
```

```js
// app/element-modifiers/noodle.js
import Ember from 'ember';

export default Ember.ElementModifier.extend({

  init() {
    this._super(...arguments);
    this._logMouseover = () => console.log('mouseover!');
  },

  didRender() {
    document.addEventListener(this.attrs.logEvent, this._logMouseover);
  },

  didUpdateAttrs(newAttrs, oldAttrs) {
    document.removeEventListener(oldAttrs.logEvent, this._logMouseover);
  },

  willDestroyElement() {
    document.removeEventListener(this.attrs.logEvent, this._logMouseover);
  }

});
```

### Positional params

Similar to the [positionalParams](http://emberjs.com/api/classes/Ember.Component.html#property_positionalParams)
API in Ember, positional params can be defined for a modifier. An example:

```hbs
<button {{noodle 'mouseover'}}></button>
```

```js
// app/element-modifiers/noodle.js
import Ember from 'ember';

const Noodle = Ember.ElementModifier.extend({

  init() {
    this._super(...arguments);
    this._logMouseover = () => console.log('mouseover!');
  },

  didRender() {
    document.addEventListener(this.attrs.mouseover, this._logMouseover);
  },

  didUpdateAttrs(newAttrs, oldAttrs) {
    document.removeEventListener(oldAttrs.mouseover, this._logMouseover);
  },

  willDestroyElement() {
    document.removeEventListener(this.attrs.mouseover, this._logMouseover);
  }

});

Noodle.reopenClass({
  positionalParams: ['mouseover']
});

export default Noodle;
```

### Rerender

An element modifier may call `this.rerender()`. This triggers the same
hook execution as would be expected from the change of an attr.

# Drawbacks

This is an easy API to abuse, and is coupled more closely to the DOM that
other helpers like components or helpers in that it deals with the setup/teardown
lifecycle of a rendered DOM node.

Element modifiers are not required to have a `-` or any distinguishing
character, thus they may conflict the variable names. For example given the
following example:

```hbs
<input {{nudge}} />
```

It must be assumed that `nudge` is an element modifier. Ember will be constrained
in that it may not add later support for `nudge` being a variable (with the
value of an HTML element-space string for example) without namespace conflicts.

# Alternatives

[RFC 100](https://github.com/emberjs/rfcs/pull/100#issuecomment-172427565) attempted
to scenario-solve event listeners across native elements, Ember component root elements, and
web components. It failed to reach a unified design that could serve all three
options. Since there does not appear to be an ideal unified solution, kicking
the challenge to use-space for a few cycles seems ideal.

Additionally, there are other uses for element modifiers beyond event listener
attachment. These may also have specific, narrow fixes.

The hooks suggested for element modifier life-cycles do not include
`didInitAttrs`, `didReceiveAttrs`, or `didUpdate`. This
may be an oversight.

Finally, instead of using component-ish hooks, we could introduce new and
bespoke hooks with whatever level of resolution we want. For example, just
a simple setup and teardown on each rerender.

We could omit the inclusion of `rerender`.

# Unresolved questions

Include attrs hooks mentioned in Alternatives?

