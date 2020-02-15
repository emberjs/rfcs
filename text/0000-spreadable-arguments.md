- Start Date: 2019-06-05
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Forwarding Component Arguments with Spreadable Arguments

## Summary

Currently it is possible to forward element attributes and modifiers into child components using the spread-like 
`...attributes` syntax. This RFC proposes adding a similar syntax and functionality for component arguments: `...@arguments`. 

## Motivation

Spreadable Arguments, or `"Splarguments"`, will allow less verbose component templates, especially where the verbosity only makes the template harder to read.
Having the ability to spread arguments would make reading deeply nested component invocations much cleaner and nicer.


For example, given the following invocation:
```hbs
<Foo @foo="bar" />
```
...and the following layout for the `Foo` component:
```hbs
<Bar ...@arguments />
```
... and the following layout for the `Bar` component:
```hbs
<div>{{@foo}}</div>
```
Ember will render the following HTML content:
```html
<div>bar</div>
```

An additional benefit of Spreadable Arguments is that it can reduce the maintenance burden of components used today.
A great use case to showcase this is when a component is wrapped by another component. Currently we have to explicitely forward every argument to the child component:

```hbs
<Foo @arg1={{@arg1}} @arg2={{@arg2}} />
```
This is very verbose and doesn't provide any benefits to the reader of the template. Having to pass all of these arguments down the component tree becomes annoying to read and hard to maintain. Imagine if we now have to add a new argument, `@arg3` to our `Foo` component. We would need to go through *every* single component invocation that is wrapping `Foo`.


An example of this wrapping pattern can be shown using a branded `Button` component. This component has many specific components that wrap the base component to provide button "types".

Button Type Components:
```hbs
<!-- success-button.hbs -->
<Button @type="success" />
```
```hbs
<!-- warning-button.hbs -->
<Button @type="warning" />
```
```hbs
<!-- error-button.hbs -->
<Button @type="error" />
```

This would be a great pattern! No need to subclass the `Button` component, you can just have a bunch of TO-components that wrap it and provide the `@type` - until you need to start adding other arguments, like `@onClick` and so on. Now you have 3 files to go update, which is a giant pain.


A possible component in the wild that could benfit from this could be 
[ember-power-select](https://github.com/cibernox/ember-power-select/blob/master/addon/templates/components/power-select.hbs).
This component has to explicitely pass on many arguments to its child components. With "splarguments" half or more of this template could be cut down.

## Detailed design

TBD

## How we teach this

This should be taught in the guides:

1. When teaching about components we should have include a section explaining how Spreadable Arguments works. A good place is here: https://guides.emberjs.com/release/components/component-arguments-and-html-attributes/#toc_arguments

2. We can teach this similarly to how Spreadable Attributes are taught today here: https://guides.emberjs.com/release/components/component-arguments-and-html-attributes/#toc_html-attributes

## Drawbacks

One drawback is the added complexity that this adds to the current component usage. It is one more thing to learn. However the potential gains for this outway this drawback.

Another drawback is that this could make certain components harder to read and understand if someone doesn't understand where the arguments are coming from. 

## Alternatives

Some alternative syntaxes:
```hbs
<!-- Pretty straightforward, and since both are keywords we can tell pretty easily what goes where -->
<SubComponent ...attributes ...arguments></SubComponent>

<!-- Using the same rule on arbitrary spreads though, we no longer have context -->
<SubComponent ...{{this.someSpecificAttrs}} ...{{this.someSpecificArgs}}></SubComponent>

<!-- Maybe prepending `@`? -->
<SubComponent ...attributes ...@arguments></SubComponent>

<!-- Seems a bit awkward putting it in front of an arbitrary array -->
<SubComponent ...{{this.someSpecificAttrs}} ...@{{this.someSpecificArgs}}></SubComponent>
```

## Unresolved questions

How will this be implemented?
Which syntax should we choose?
Should this encompass arbitrary spreads of objects or just arguments?
