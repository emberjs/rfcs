---
Start Date: 2019-01-18
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/435
Tracking: https://github.com/emberjs/rfc-tracking/issues/9

---

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

1. It is legal to apply modifiers to angle bracket component invocations, i.e.

   ```hbs
   {{!-- this is legal --}}
   <MyButton {{action this.onClick}}>Click Me!</MyButton>
   ```

2. Element modifiers can be applied to the underlying HTML element(s), along
   with any HTML attributes, using the splattributes syntax.

   ```hbs
   {{!-- this apply any modifiers in addition to HTML attributes --}}
   <a ...attributes>{{yield}}</a>
   ```

3. In addition, the splattributes syntax can be used to forward HTML attributes
   and element modifiers to subsequent angle bracket component invocations.

   ```hbs
   {{!-- this is also legal, does the same as the above --}}
   <InternalButton ...attributes>{{yield}}</InternalButton>
   ```

This allows the end-users to retain some control over DOM event handling and
other HTML concerns (such as CSS and ARIA roles/accessibility concerns) when
invoking components.

Fundamentally, element modifiers simply enable more fine-grained customization
of an HTML element, on top of what one could accomplish with HTML attributes.
If it is possible to configure the `class` and `aria-role` attributes of a
component's HTML element, it should also be possible to extract them into a
custom element modifier.

It is also adventageous to allow modifiers like `action` to work consistently,
whether the invocation happens to be an HTML element or a component. This allow
features like the [element helper](https://github.com/emberjs/rfcs/pull/389) to
compose better.

For these reasons, we believe it is important and consistent to allow these
interactions.

## Detailed design

From Glimmer VM's perspective, the foundation for these features are already
in-place. Specifically, when applied on an angle bracket invocation, HTML
attributes and element modifiers are collected into an internal block, and the
splattributes syntax simply yields back to that block. Similarly, when applying
the splattributes to another angle bracket invocation, it simply fowards the
block recurrsively. This feature is only currently gated by a precautionary
"compile time error" which can be easily removed once this RFC is accepted.

As laid out in the [modifier manager RFC](https://github.com/emberjs/rfcs/pull/373),
the `createModifier` hook is called in the order they appear in the template.
This means that given the following invocation:

```hbs
<MyComponent {{bar}} />
```

And the following template for `MyComponent`:

```hbs
<div {{foo}} ...attributes {{baz}} />
```

The creation order will be `{{foo}}`, `{{bar}}`, `{{baz}}`. However, the RFC
only provide relative timing guarentees for `createModifier`, and notably _not_
for `installModifier` and `updateModifier` where most of the interesting work
happen (`createModifier` does not receive the element). Therefore, in practice,
it is both not very useful to rely on this timing guarentee, nor is it a good
idea.

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

With the changes proposed in this RFC, it becomes more important to emphasize
that element modifier is a "sharp tool". As with lifecycle hooks in the classic
`Ember.Component`, element modifier is an escape valve from the declarative,
pure and functional world of Handlebars templates, into the messy world of
imperative code, shared states and mutability. While they are very flexible,
that flexibility comes at a cost. When used incorrectly, they can easily leak
state, stomp over each other and causes problems in the app.

Therefore, when authoring element modifiers, it is important to be a "good
citizen", keeping in mind that the underlying HTML element is "shared" among
any bound attributes in the template and other element modifiers. For example,
it is probably a bad idea to prevent event propagation from within an element
modifier, as it may break other modifiers that are listening to the same DOM
event.

This problem is not new, as it is already possible to have multiple element
modifiers attached to the same HTML element. However, when intermediate
components are involved, this could become very difficult to notice.

Therefore, it is even more important to teach and encourage users to author
element modifiers that play well with each other to allow the kind of
composition proposed in this RFC to work at scale.

On the flip side, installing element modifiers on extenal components (i.e.
those that came from outside the app, such as those provided by addons) is also
a somewhat fragile act as it pierces through an encapsulation boundries. Very
generic modifiers like `{{action}}` and `{{on}}` are unlikely to cause problems,
but more special-purpose ones may not be appropiate, unless they are sanctioned
by the component authors.

This is already a risk with splattributes in general, as there are plenty of
context-specific HTML attributes. However, allowing element modifiers here is
going to increase the risk as the operations they perform are hidden further
away.

## Drawbacks

The main drawback is the added risk of breaking encapsulation boundries of
components. Specifically, because the element modifiers have access to the raw
underlying HTML element, they may inadvertently depend upon details about the
element (it is of a particular type, has certain attributes or properties set,
etc), beyond what was intended by the component author as a public API. If this
turned out to be a wide-spread problem, it can be mitigated by adding linting
rules to the template linter.

Separately, as proposed, this API does not allow the element modifiers to "see"
any intermediate components, only the final HTML element. If this turned out to
be useful, we can consider introducing it as an optional capability in future
extensions.

## Alternatives

We can disallow using element modifiers on components, as well as using
splattributes to forward HTML attributes on child component invocations.

## Unresolved questions

None.
