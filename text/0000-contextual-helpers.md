- Start Date: 2016-02-7
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The goal of this RFC is to increase composability opportunities in Ember by allowing helpers
to be invoked and curried in a similar way as how actions and components can be dynamically
be invoked now.

# Motivation

Since Ember 2.3, released over a year ago, the framework has enjoyed a feature commonly
referred to as **contextual components** that allows components to be dynamically invoked by name
and also to be wrapped along with arguments and passed as regular values to later on be invoked.

Example from the blog post of the 2.3 release:

```hbs
{{#alert-box as |box|}}
  Danger, Will Robinson!
  <div style="float:right">
    {{#box.close-button}}
      It's just a plain old meteorite.
    {{/box.close-button}}
  </div>
{{/alert-box}}
```

This features allows the creation of very nice domain-specific APIs in components. Some examples
of this in the wild can be found in [ember-basic-dropdown](https://github.com/cibernox/ember-basic-dropdown),
[ember-form-for](https://github.com/martndemus/ember-form-for) and [ember-power-calendar](https://github.com/cibernox/ember-power-calendar).

However, the same feature is not yet supported for helpers and it would be equally useful for
exposing helpers with prepopulated arguments or options to provide better APIs.

# Detailed design

The design of this feature should be as similar as possible to the **contextual components** feature,
so users feel confortable with it immediatly.

Therefore, the natural option would be the creation of a `helper` helper that, depending on the context,
invokes a helper by its name or creates a closure helper to pass it around.

However, there is an ambiguity with this approach that is not present with components. I'm going to expose
the problem first, since it's going to drive the entire design of the feature.

```hbs
{{my-component format-date=(helper "moment-format")}}
```

In the example above it's not clear the intention of the developer. The developer's intention
can be a few things:

- Immediatly evaluate the helper and pass the value of its evaludation to the component
- Create a closure helper and pass it to the component, so it can be called later.

Note that this ambiguity is not present when the `helper` helper is called in the
context of html. By example

```hbs
<strong>My name is {{helper 'capitalize-str' user.name}}</strong>
<button title={{helper 'capitalize-str' user.name}}></button>
```

In those examples above the only option can be immediatly evaluate.

How is this problem solved?

#### Use a different helper for creating closure helpers and to evaluate helpers

To remove the ambiguity, there are two different keywords for the two different features.

Ideas include `helper` to create the closure helper and `invoke-helper` to invoke it.
Another possible names borrowed from Ruby conventions can be `helper` (closure creator) and `helper!` (invocation).

Examples:

```hbs
<strong>My name is {{invoke-helper 'capitalize-str' user.name}}</strong>
<button title={{invoke-helper 'capitalize-str' user.name}}></button>
<strong>My name is {{helper 'capitalize-str' user.name}}</strong> <!-- Invalid invocation -->
<button title={{helper 'capitalize-str' user.name}}></button> <!-- Invalid invocation -->

{{my-component format-date=(helper "moment-format")}} <!-- Creates closure helper -->
{{my-component format-date=(invoke-helper "moment-format")}} <!-- Invokes helper -->
```

The obvious drawback of this alternative is having two new keywords in Ember instead of one.

Ideally, since the idea of wrapping a component and a helper is very similar, I think that 
this keyword should be used for both components and helpers and replace the existing 
overloaded `component` keyword.

# How We Teach This

Given the parallelism with closure components it's not going to be hard to teach. It will follow the usual
process of documentation in the API and in the guides.

# Drawbacks

Every dynamic feature has some runtime cost. I've heard about some proposals to limit the possible
components that the `{{component}}` helper can invoke so the dependency resolution can be done statically.
Whatever solution is designed for the component helper must also be take into consideration for the `{{helper}}` helper.

# Alternatives

The RFC already discusses the alternative paths since they are the main subject of the proposal.

# Unresolved questions

To be determined.
