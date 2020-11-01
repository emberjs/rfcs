---
Start Date: 2018-10-14
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/389
Tracking: https://github.com/emberjs/rfc-tracking/issues/42

---

# Dynamic tag names in glimmer templates.

## Summary

With the transition from inner-html semantics to outer-html semantics in components, we lost one feature: Being
able dynamically define the tag name of components.

I think it was an useful feature and we should find a way to bring it back.

## Motivation

Although not something we use every day, there is a need for some components to have a dynamic tag name.

This is most often used for certain low-level _presentational_ components.

Take for instance a component named `<Panel>` that is used to encapsulate some presentation concerns.
The template of that component could be like this:

```hbs
<div class="pt-10 pb-10 ps-20 box-shadow" ...attributes>
  {{yield}}
</div>
```

For accessibility and semantic reasons, sometimes a `<div>` may not be the best kind of tag.
We might want the panel to be a `<section>` element, or an `<aside>` when it's content is somewhat unrelated
with the rest of the page. Or maybe a `<legend>` if it's and the end of a form.

With the `element` helper proposed in this RFC, this can be accomplished with something like this:

```hbs
{{#let (element @tagName) as |Tag|}}
  <Tag class="pt-10 pb-10 ps-20 box-shadow" ...attributes>
    {{yield}}
  </Tag>
{{/let}}
```

## Detailed design

We propose to add a new `element` helper that takes a single positional argument.

* When passed a non-empty string it generates a contextual component that, when invoked, renders an element with the same tag name as the passed string, along with the passed attributes (if any), modifiers (if any) and yields to given block (if any).
* When passed an empty string, it generates a contextual compoment that, when invoked, yields to the given block without wrapping it an element and ignores any passed modifiers and attributes.
* When passed `null` or `undefined`, it will return `null`.
* When passed any other values (e.g. a boolean or a number), it will result in a development mode assertion.

Example:

```hbs
{{#let (element @htmlTag) as |Tag|}}
  <Tag class="my-element" {{on "click" this.tagClicked}}>Hello</Tag>
{{/let}}

{{!-- when @htmlTag="button" --}}
<button class="my-element" {{on "click" this.tagClicked}}>Hello</button>

{{!-- when @htmlTag="" --}}
Hello

{{!-- when @htmlTag=null or @htmlTag=undefined, it renders nothing --}}

{{!-- when @htmlTag=true or @htmlTag=1, it throws in development mode --}}
```

Unlike ids, classes or other attributes, the tag name of DOM element cannot be changed in runtime.

To help the user understand that changing the tag of an element in runtime is an expensive operation,
the syntax is intentionally chosen to express that changes in the tag name will turn down the given block
and recreate it again.

Having dynamic tag names can also open the door to possible XSS vulnerabilities if developers allow user-input
to become tag names (e.g. `xlink:`) . To prevent that, this helper will throw an error for any tag name containing anyting
but lowercase letters and dashes.

A working proof of concept of this approach has been created in https://github.com/tildeio/ember-element-helper

Shall this RFC me accepted, that helper would be ported to Ember.js itself, perhaps making it more efficient
on the process.

## How we teach this

_It's vitally important that developers have awareness the negative impact that misuse of dynamic tags can have on the accessibility of an Ember application._

While dynamic tags enable a great deal of flexibility in components, it's essential to recognize that these can also negatively impact accessibility. Developers should be keenly aware of the appropriate role that should be applied per HTML element (as specified in the [WAI-ARIA specification](https://www.w3.org/WAI/PF/aria/roles)), and ensure that the role is also updated, if necessary, as the tag name is changed. In some cases, the use of native, semantic HTML elements may eliminate the need to apply a role at all, so developers should consult the WAI-ARIA specification until they are certain of the correct use for their specific use cases.

This new helpers will be added to the list of [built-in helpers](https://emberjs.com/api/ember/release/classes/Ember.Templates.helpers) along
with its cousins `component`, `array`, `each`, etc...

Some meaningful example for the docs could look like this (extracted from a real use case):

```hbs
{{!-- sidebar.hbs, a template-only component --}}
{{#let (element (or @htmlTag "aside")) as |Tag|}}
  <Tag ...>...</Tag>
{{/let}}
```

```hbs
<Sidebar @htmlTag="nav">...</Sidebar>
```

Since this feature is not very commonly used, it should not be mentioned in the more beginner-friendly portion
of the guides. However, the _Components > Customizing a Component's Element_ section is a perfect fit for this.

## Drawbacks

Admittedly this syntax is somewhat convoluted, as it involves using the `let` helper, the new `element`
helper that yields a contextual component that is then invoked using angle-bracket syntax.

This syntax is intentional to make clear that any change in the tag name would yield a new contextual component,
effectively tearing down the previous one before rendering the new one, but I can see perceiving this
as unnecessarily complex.

## Alternatives

We can decide not to do anything and leave this problem to be solved in user space, as it can be
solved using only public APIs.

We can also propose other alternative syntaxes. For instance, we could have a special built-in component for this:

```hbs
<DynamicElement @tagName={{@tagName}} class="some-class" ...attributes>
</DynamicElement>
```

## Unresolved questions

---
