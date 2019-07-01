- Start Date: 2019-06-05
- Relevant Team(s): Ember.js
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Forwarding Component Arguments with Spreadable Arguments

## Summary

Currently it is possible to forward element attributes and modifiers into child components using the spread-like 
`...attributes` syntax. This RFC proposes adding a similar syntax and functionality for component arguments: `...arguments`. 

## Motivation

Spreadable Arguments, or `"Splarguments"`, will allow less verbose component templates, especially where the verbosity only makes the template harder to read.
Having the ability to spread arguments would make reading deeply nested component invocations much cleaner and nicer.


For example, given the following invocation:
```hbs
<Foo @foo="bar" />
```
...and the following layout for the `Foo` component:
```hbs
<Bar ...arguments />
```
... and the following layout for the `Bar` component:
```hbs
<div>{{@foo}}</div>
```
Ember will render the folliwing HTML content:
```html
<div>bar</div>
```

An additional benefit of Spreadable Arguments is that is can reduce the maintenance burden of components used today.
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


A possible example in the wild could be 
[ember-power-select](https://github.com/cibernox/ember-power-select/blob/master/addon/templates/components/power-select.hbs).
This component has to explicitely pass on many arguments to its child components. With "splarguments" half or more of this template could
be cut down.

## Detailed design

> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

## How we teach this

> What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users
at any level?

> How should this feature be introduced and taught to existing Ember
users?

## Drawbacks

> Why should we *not* do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
