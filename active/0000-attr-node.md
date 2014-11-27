- 2014-11-26
- RFC PR:
- Ember Issue:

# Summary

HTMLBars will introduce a few new syntaxes to Ember. Supporting these
means two challenges need to be addressed:

* First, there are a variety of different implications HTMLBars
  attributes may have. In general HTMLBars attribute behave like
  DOM element properties and have two states: Unquoted (thus a raw
  value) and quoted (usually a string). There are exceptions that
  need to be accounted for, but in general setting properties is
  a good path forward.
* Second, Ember needs a general construct that mirrors the concept
  of an `AttributeNode` on the DOM. This construct I'm going to call
  and `AttrNode`. You might think of it as equivalent to what a `View`
  is for a DOM element.

This RFC will lay out a justification for the the property setting
approach and illustrate several examples and edge cases. It will outline
the basic facets of the `AttrNode` construct, which will remain an
internal implementation detail of Ember for the immediate future.

# Motivation

Let us consider several HTMLBars syntax examples, the expected behavior,
and the challenge of having that behavior occur.

```hbs
<div id="{{divId}}"></div>
```

* First, this is the most basic of possible syntaxes. The `id` is
  obviously intended to be a string, both because the developer has
  quoted it and `id`s in HTML are strings.
* Easy-peasy my man.

```hbs
<div id={{divId}}></div>
```

* Here the user has not explicitly passed a string. However, we can
  imply that the intent is to set a string since dom `id`s are always
  strings.
* An un-quoted value means the user is no longer explicitly passing a
  string.

```hbs
<input type="date" value={{today}}>
```

* Starting to get complex. Because the value is not quoted, what type
  should `today` be?
* Some un-quoted values may not be strings.

```hbs
<input disabled={{isDisabled}}>
```

* Boolean attributes are tricky, but really no more tricky than dates.
  A boolean value is normally set to the `disabled` attribute's corresponding
  property, and it could be presumed that the intent is the same here.
* Un-quoted values will likely match their attribute's corresponding
  property behavior.

```hbs
<svg viewBox={{viewBoxSize}}></svg>
```

* The `viewBox` attribute has no helpful, writable corresponding property. In
  fact several SVG attributes behave the same way.
* Some HTMLBars attributes won't map well to properties.

```hbs
<input maxlength={{length}}></input>
```

* The `maxlength` attribute maps to a property with the name `maxLength`.
  Several other HTML attributes do this.
* Some HTMLBars attributes map to properties with different names.

```hbs
<div data-color={{color}}></div>
```

* The `data-color` attribute doesn't have a corresponding property
  directly on a DOM element.
* Some HTMLBars attributes don't map directly to properties.

```hbs
<div class="{{size}} blue {{length}}"></div>
```

* The ideal internal for Ember classes is `classList`, a newer API
  that represents classes as a set. This API is faster than any
  alternative and supports SVG. The values for this attribute (to
  HTMLBars) are two streams and a string. The user implication that
  a string is passed (via quoting) should be disregarded.
* Some HTMLBars attributes will have wildly different implementation
  behavior. `class` is an extreme examples of this, `style` is another.

To support these different interactions, the abstraction of the
`AttrNode` will be introduced.

### Looking forward

Unquoted values being passed as "raw" objects (not converted to string),
also prepares Ember for eventual Ember component syntaxes:

```hbs
<audio-player toggle={{action togglePlayback}}>
```

```hbs
<ember-input value={{mut name}}>
```

The `action` and `mut` helpers can return `AttrNode`s or some object
that attribute nodes understand how to interact with. This idea
informs this RFC, but solving it in detail is not proposed here.

# Detailed design

Here I'm going to flip this discussion on its head and lead with  `AttrNode`
as a thing. `AttrNode` will helper us solve some of the challenges
identified, and some will need to be solved in the HTMLBars DOMHelper.

In HTMLBars the `attribute` hook is called with these arguments:

```
element, attributeName, quoted, view, parts, options, env
```

Three of these are sufficient to determine which kind of challenges need to
be tackled:

```
attributeName, element, quoted
```

These can be passed to a `attrNodeFor` helper that will return a class or
instance matching the `AttrNode` duck type.

`AttrNode` objects themselves will likely need four arguments:

```
element, attributeName, parts, dom
```

The lifecycle of a simple `AttrNode`, say for an unquoted string attribute,
is pretty simple:

* If the value of the attribute is a stream, subscribe to the stream. If
  the value changes then flag the node as dirty and schedule a render.
* Then the render occurs there maybe have been several value notifications. The
  implementation should flush the dirty state.
* After the subscription is made, render immediately. The render step should
  be agnostic about if this is an initial value being rendered or an update.

I propose a simple internal construct:

```js
function AttrNode(element, attributeName, attributeValue, dom) {
  // Setup the object, call renderIfNeeded
  //
  // For most cases, attributeValue will be a single value or stream
  // in an array. However passing the array of values is important,
  // since complex nodes like class will need to utilize the
  // individual streams. A 'concat' node class can make quoted
  // nodes simple to implement.
}

AttrNode.prototype.renderIfNeeded = function renderIfNeeded(){
  // 1. Mark the node dirty
  // 2. Schedule render into runloop
  //
  // The scheduled render task:
  // 1. Don't proceed if the node is clean
  // 2. Mark the node clean
  // 3. Read the attributeValue stream
  // 4. If the currentValue and stream value are different,
  //    set the currentValue, lastValue and render hook
};

AttrNode.prototype.render = function render(){
  // In this hook currentValue and lastValue are present.
  // Render should update the element (using the DOMHelper)
  // as appropriate.
};
```

This manages the rendering lifecycle of attribute nodes.

# Drawbacks

The general leaning toward properties, and syntax, seems to be well
considered. In a pragmatic sense there are bound to be edge cases and
browser quirks that will need to be handled.

There might be a design to replace the attribute nodes with something
stateless. Stream values themselves are cached, and the same
edge-node-limit-dom-access approach could in theory be achieved
functionally.

# Alternatives

Two obvious alternatives considered in detail are Angular and React.

In **Angular 2.0**, [a new prop/attr/event syntax](http://www.beyondjava.net/blog/angularjs-2-0-sneak-preview-data-binding/)
is being introduced.

Setting an attribute just like setting an HTML attribute:

```html
<pui-tab title="What a nice tab!">
```

Properties are flagged with the `[]` syntax:

```html
<input [disabled]="controller.isInputDisabled">
```

Angular is limited by it's HTML templating here. The value must be quoted
to have complex content, where as in HTMLBars it is easier to bend the
rules to introduce literal values: `disabled={{controller.isInputDisabled}}`.

Events are out of our immediate purview in this RFC, but for completeness
note Angular's syntax:

```html
<button (click)="hide()">hide image</button>
```

In Ember we expect to again use the literal syntax to pass `hide` (a
function) wrapped as an action helper to the `click` or `on-click`
property (to be decided by the Ember HTMLBars component syntax).

**React's JSX** has its own [property syntax](http://facebook.github.io/react/docs/jsx-in-depth.html),
one that diverges from traditional HTML by focusing entirely on properties
instead of attributes. This means the templates are well prepared for
use with components, but also that JSX must maintain a large whitelist of
special cases such as [supported tags](http://facebook.github.io/react/docs/tags-and-attributes.html)
and [some HTML attributes](http://facebook.github.io/react/docs/jsx-gotchas.html).

In general we would prefer to have Ember templates be as close to HTML
as possible, without requiring developers to learn a new set of property
names replacing the attribute names they already know.

# Unresolved questions

This design does not address components at all. It limits itself to an initial
implementation of the HTMLBars syntax for attributes.

There is a spike of significant depth [in PR #9721](https://github.com/emberjs/ember.js/pull/9721),
though it is not in sync with this rfc. During implementation, it is suggested
that the current `bind-attr` backend also be refactored toward the `AttrNode`
construct.
