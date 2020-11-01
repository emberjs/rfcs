---
2014-11-26: 
RFC PR: https://github.com/emberjs/rfcs/pull/24
Ember Issue: 

---

# Summary

Unlike Handlebars, HTMLBars parses HTML as it parses a template.
Bound attributes are one syntax now possible.

For example, this variable `color` is bound to set a class:

```hbs
<div class="{{color}}"></div>
```

Though traditional HTML attribute syntax should be preserved (using
`class` and not `className` for example), the default path will be
to set attributes as properties on the DOM node.

However this happy path has several important exceptions, and results
in a few strange edge cases. This rfc will go into detail about the
expected behavior without talking about the implementation of attribute
on the Ember rendering pipeline.

# Motivation

`{{bind-attr` is a verbose syntax and difficult for new developers to
understand.

# Detailed design

Given a use of bound attributes:

```hbs
<input type="checkbox" checked={{isChecked}}>
```

There are three important inputs:

* The element (`tagName`, `namespaceURI`)
* The attribute name
* The value (literal or stream)

The following described the algorithm for updating the attribute/property
value on an element.

1. If the element has an SVG namespace, use `setAttribute`. Setting SVG attributes
   as properties is not supported.
2. If the attribute name is `style`, use `setAttribute`.
3. Normalize the property name as described in `propertyNameFor` below. If a normalized
   name is returned, set that property on the element (`element[normalizedPropName]`).
   If it is not returned, set with `setAttribute`.

`propertyNameFor` is a normalization setup for attribute names that takes the element
and attribute name as input.

1. Build a list of normalized properties for the passed element `normalizedAttrs[element.tagName][elementAttrName.toLowerCase()] = elementAttrName`
2. Fetch the normalized property name from this list `normalizedAttr = normalizedAttrs[element.tagName][passedAttrName.toLowerCase()]`
3. Return this normalized attr. If an `attrName` is did not normalize to a property (for example `class`), null is returned

### Acknowledged edge cases

* Boolean attrs with blank string won't work like they would in HTML: `<input disabled="{{blankString}}">` would be false
* Some selectors may not work as expected. `<input value="{{color}}">` will not result in a working `[value=red]` selector

# Drawbacks

None.

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

* How do we deal with `on*` attributes?
* Should we do anything special about generic element properties like `<div outerhtml={{lol}}></div>`?
* Should HTMLBars unbound attributes use the same alorithm?

There is a spike of significant depth [in PR #9721](https://github.com/emberjs/ember.js/pull/9721)
and a followup [in PR #9977](https://github.com/emberjs/ember.js/pull/9977).
