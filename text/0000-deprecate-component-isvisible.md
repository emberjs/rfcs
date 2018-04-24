- Start Date: 2018-03-28
- RFC PR:
- Ember Issue:

# Summary

The aim of this RFC is to deprecate the component `isVisible` attribute 
is not used by ember internally and left undefined unless manually set.
It's poorly documented and component visibility it better managed in 
template space rather than JS.

# Motivation

`isVisible` is super legacy and not entirely necessary Component visbility 
is better handled in tempalte space apposed to the JS alternative. Using 
this attribute to toggle component visibility introduces bugs with `StyleBindingReference` 
not updating. Currently, it adds `display: none` as an inline style which removes 
all other inline styles attached to an element. It is an additional way of 
hiding components that with its removal will reduce confusion on which approach
to take when performing said functionality.

# Transition Path

In cases where the `isVisible` property is used we provide a deprecation warning
with a link to the deprecation guide which states why it was deprecated and/or the
options available to hide the component. As it is a public API it will be removed
in the next major version release (4.0).

There are several options available to hiding elements 
such as `<div hidden={{boolean}}></div>`(hidden is valid for all elements
and is semantically correct) or wrapping the component in a template
conditional `{{#if}}` statement which do not interfere with
the `StyleBindingReference`. Components `classNames` an `classNameBindings`
could also be used to add hidden classes.

# How We Teach This

The `isVisible` attribute is rarely used so deprecating in a future blog post
would be sufficient. It will need to be removed from the API docs. It would be
beneficial to add documentation on hiding components to the Ember guides with the
conditional handlebar helper.
`{{#if showComponent}}`
  `{{component}}`
`{{/if}}`

Alternatively, with the now widely supported HTML hidden attribute using a simple
`<div hidden={{isHidden}}></div>` where isHidden can be toggled.

# Alternatives

An alternative option would be to to keep `isVisible`.
