---
Start Date: 2018-03-28
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/324
Tracking: https://github.com/emberjs/rfc-tracking/issues/22

---

# Summary

The aim of this RFC is to deprecate the component's `isVisible` property. 
It is not used by Ember internally and left undefined unless manually set.
It's poorly documented and component visibility it better managed in 
template space rather than JS.

# Motivation

Setting the isVisible property on a component instance as a way to toggle
the visibility of the component is confusing. The majority of its usage
predates even Ember 1.0.0, and modern Ember applications already completely
avoid using isVisible in favor of simpler conditionals in the template
space.

In addition, when `isVisible` is used today it often introduces subtle (and
difficult to track down) bugs due to its interaction with the `style`
attribute (toggling `isVisible` clobbers any existing content in `style`).

Simply put, removing `isVisible` will reduce confusion amongst users.

# Transition Path

Whenever `isVisible` is used a deprecation will be issued with a link to 
the deprecation guide explaining the deprecation and how to refactor in order
to avoid it.

Given that `Component#isVisible` is a public API, deprecating now would
schedule for removal in the next major version release (4.0).

There are several options available to hiding elements 
such as `<div hidden={{boolean}}></div>`(hidden is valid for all elements
and is semantically correct) or wrapping the component in a template
conditional `{{#if}}` statement. Components `classNames` and `classNameBindings`
could also be used to add hidden classes.

# How We Teach This

The `isVisible` property is rarely used, the deprecation along with a mention
in a future blog post would be sufficient.

We should consider adding documentation on hiding components to the Ember
guides with the conditional handlebar helper or via the widely supported `hidden`
attribute.

```hbs
{{#if showComponent}}
  {{component}}
{{/if}}

{{! or }}
<div hidden={{isHidden}}></div>
```

# Alternatives

An alternative option would be to to keep `isVisible`.
