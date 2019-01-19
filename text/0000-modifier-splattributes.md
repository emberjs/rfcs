- Start Date: 2019-01-18
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Forwarding Element Modifiers with "Splattributes"

## Summary

This is a small amendment to
[RFC #311 "Angle Bracket Invocation"](https://emberjs.github.io/rfcs/0311-angle-bracket-invocation.html)
and [RFC #373 "Element Modifier Manager"](https://emberjs.github.io/rfcs/0373-Element-Modifier-Managers.html)
to clarify how the "splattributes" feature interact with element modifiers.

## Motivation

RFC #311 introduced the angle bracket component invocation feature. Aside from
the syntatic differences, the angle bracket invocation syntax enabled passing
HTML attributes to components, which can then be applied the underlying HTML
element(s) in the component's layout using the `...attributes` "splattributes"
syntax.

For example, given the following invocation:

```hbs
<FooBar class="foo-bar" />
```

...and the following layout for the `FooBar` component:


```hbs
<div ...attributes>foo bar!</div>
```

Ember will render the following HTML content:

```html
<div class="foo-bar">foo bar!</div>
```

See the [HTML Attributes section](https://emberjs.github.io/rfcs/0311-angle-bracket-invocation.html#html-attributes)
of RFC #311 for more information on this feature.

On the other hand, RFC #373 introduced the element modifier manager feature.
This enabled Ember developers to define custom element modifiers, similar to
the built-in `{{action}}` modifier that ships with Ember.

This feature can be quite useful for encapsulating, among other things, DOM
event handling and accessibility concerns. For example:

```hbs
<a {{on click=(action this.onClick)}} {{act-as "button"}}>Click Me!</a>
```

While these features are both very useful on their own, they can be combined
to enable powerful abstraction and composition patterns. Unfortunately, the
two RFCs did not explicitly describe how these features would interact with
each other. This RFC proposes three admenments to clarify their relationship:

1. It is legal to apply modifiers to angle bracket component invocations.

2. Element modifiers can be applied to the underlying HTML element(s), along
   with any HTML attributes, using the splattributes syntax.

3. In addition, the splattributes syntax can be used to forward HTML attributes
   and element modifiers to subsequent angle bracket component invocations.

This allows the end-users to retain some control over DOM event handling and
other HTML concerns (such as CSS and ARIA roles/accessibility concerns) when
invoking components.

Fundamentally, element modifiers simply enable more fine-grained customization
of an HTML element, on top of what one could accomplish with HTML attributes.
If it is possible to configure the `class` and `aria-role` attributes of a
component's HTML element, it should also be possible to extract them into a
custom element modifier. Therefore, we believe it is important and consistent
to allow these interactions.

## Detailed design

From Glimmer VM's perspective, the foundation for these features are already
in-place. Specifically, when applied on an angle bracket invocation, HTML
attributes and element modifiers are collected into an internal block, and the
splattributes syntax simply yields back to that block. Similarly, when applying
the splattributes to another angle bracket invocation, it simply fowards the
block recurrsively. This feature is only currently gated by a precautionary
"compile time error" which can be easily removed once this RFC is accepted.

## How we teach this

This should be taught in the guides:

1. When teaching angle bracket invocations, we should mention that HTML
   attributes and modifiers, in addition to named arguments, can be passed to
   components. Some examples would be passing `class`, `aria-role` and the
   built-in `action` modifier.

2. When teaching how to author component layouts, we should introduce the
   splattributes syntax and explain why it is a good practice to include it on
   the primary element(s) in the layout, in order to allow custom styling and
   accessibility management by the end-user.

3. When teaching advanced component composition patterns, we can introduce the
   concept of "components that invokes other components". This would be a good
   place to explain how the splattributes can be used to forward both HTML
   attributes as well as modifiers to child components.

4. When teaching element modifiers, we can give use cases of refactoring common
   set of HTML attributes (e.g. classes that goes together with aria-roles)
   into named element modifiers (e.g. `{{act-as "button"}}`).

## Drawbacks

As proposed, this API does not allow the element modifiers to "see" any
intermediate components, only the final HTML element. If this turned out to be
useful, we can consider introducing it as an optional capability in future
extensions.

## Alternatives

We can disallow using element modifiers on components, as well as using
splattributes to forward HTML attributes on child component invocations.

## Unresolved questions

None.
