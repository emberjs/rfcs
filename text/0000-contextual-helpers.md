- Start Date: 2016-10-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The goal of this RFC is to allow for better helper composition by allowing helpers
to be invoked and curried in a similar way as how actions and components can be dynamically
be invoked now.

# Motivation

Since Ember 2.3, released in January of 2016, the framework has enjoyed a feature commonly
referred a **contextual components** that allows components to be dynamically invoked by name
and also to be wrapped along with arguments and passed as regular values to later on be invoked.

Examples from the blog post of the 2.3 release:

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

This features allows to create very nice domain-specific APIs in components. Some examples
of this in the wild can be found in [ember-basic-dropdown](https://github.com/cibernox/ember-basic-dropdown),
[ember-form-for](https://github.com/martndemus/ember-form-for) and [ember-form-for](https://github.com/cibernox/ember-power-calendar).

However, the same feature is not yet supported for helpers and it would be equally useful for
exposing helpers with prepopulated arguments or options to provide better APIs.


# Detailed design

The design of this feature should be as similar to the **contextual components** feature as
possible so users feel confortable with it immediatly.

Therefore the natural option would be the creation of a `helper` helper that depending on the context
invokes a helper by its name or creates a closure helper to pass it around.

However there is an ambiguity with this approach that is not present with components. I'm going to explose
the problem first since it's going to drive the entire design of the feature.

```hbs
{{my-component format-date=(helper "moment-format")}}
```

In the example above it's not clear the intention of the developer. The developer might be
trying:

- Immediatly evaluate the helper and pass the value of its evaludation to the component
- Create a closure helper and pass it to the component, so it can be called later.

Note that this ambiguity is not present when the `helper` helper is called in the
context of html. By example

```hbs
<strong>My name is {{helper 'capitalize-str' user.name}}</strong>
<button title={{helper 'capitalize-str' user.name}}></button>
```

In those examples above the only option can be immediatly evaluate.

In the proposed syntax of glimmer-components the ambiguity doesn't exists when passing attributes
but it does when passing properties:

```hbs
<my-component class={{helper 'dasherize-str' className}}></my-component> <!-- Immediate evaluation -->
<my-component @class={{helper 'dasherize-str' className}}></my-component> <!-- Unclear. Can be evaluation or closure components-->
```

How is this problem solved?

There is three possible options

#### Use a different helper for creating closure helpers and to evaluate helpers

This is the most evident aternative. To remove the ambiguity, there is two different helpers for the two different
features.

Ideas include `helper` to create the closure helper and `invoke-helper` to invoke it.
Another possible names borrowed from ruby conventions can be `helper` (closure creator) and `helper!` (invocation).


Examples:

```hbs
<strong>My name is {{invoke-helper 'capitalize-str' user.name}}</strong>
<button title={{invoke-helper 'capitalize-str' user.name}}></button>
<strong>My name is {{helper 'capitalize-str' user.name}}</strong> <!-- Invalid invocation -->
<button title={{helper 'capitalize-str' user.name}}></button> <!-- Invalid invocation -->

{{my-component format-date=(helper "moment-format")}} <!-- Creates closure helper -->
{{my-component format-date=(invoke-helper "moment-format")}} <!-- Invokes helper -->
```

#### Wrap in parameters to force imediate execution

This leaves ember with only on `helper` helper that by default is a closure creator but when wrapped in an extra pair or parens
behavies like `invoke-helper` did on the previous section.

Examples:

```hbs
<strong>My name is {{(helper 'capitalize-str' user.name)}}</strong>
<button title={{(helper 'capitalize-str' user.name)}}></button>
<strong>My name is {{helper 'capitalize-str' user.name}}</strong> <!-- Invalid invocation -->
<button title={{helper 'capitalize-str' user.name}}></button> <!-- Invalid invocation -->

{{my-component format-date=(helper "moment-format")}} <!-- Creates closure helper -->
{{my-component format-date=((helper "moment-format"))}} <!-- Invokes helper -->
```

This is dryer but probably too subtle, and double paranthesis have been historycally a syntax error in Ember.

#### Use an option to disambiguate

This is not very different from the first strategy but instead of using a different helper it passes a special option
to change the default behaviour.

For the next example the default behaviour will be immediate-execition helper and the opt-in will be creating a closure
helper (`closure=true`).


```hbs
<strong>My name is {{helper 'capitalize-str' user.name}}</strong>
<button title={{helper 'capitalize-str' user.name}}></button>
<strong>My name is {{helper 'capitalize-str' user.name closure=true}}</strong> <!-- Invalid invocation -->
<button title={{helper 'capitalize-str' user.name closure=true}}></button> <!-- Invalid invocation -->

{{my-component format-date=(helper "moment-format" closure=true)}} <!-- Creates closure helper -->
{{my-component format-date=(helper "moment-format"))}} <!-- Invokes helper -->
```

#### Static code or runtime analysis?

I'm unsure if this is feasible, but with glimmer's cross-template optimizations it might be possible to
detect how the passed component is used in the context where it is passed and the desambiguation is done
by the engine.

Examples:

```hbs
<strong>My name is {{helper 'capitalize-str' user.name}}</strong>
<button title={{helper 'capitalize-str' user.name}}></button>

{{my-component-a format-date=(helper "moment-format")}} <!-- We don't know yet -->
{{my-component-b format-date=(helper "moment-format")}} <!-- We don't know yet -->
{{yield (hash format-date=(helper "moment-format"))}} <!-- We don't know yet -->

<!-- Inside my-component-a.hbs -->
<strong>My name is {{format-date}}</strong> <!-- The engine detects that is used in html-context and therefore it is considered an evaluation -->

<!-- Inside my-component-b.hbs -->
<strong>My name is {{helper format-date date locale="fr"}}</strong> <!-- The engine detects that is used as a closure helper-->

<!-- On the yielded context -->
{{#my-yielder as |api|}}
  {{api.format-date}} <!-- Invocation with dot-syntax means it's still ambiguous??-->
{{/my-yielder}}
```

# How We Teach This

Given the parallelism with closure components it's not going to be hard to teach. It will follow the usual
process of documentation in the API and in the guides.

# Drawbacks

Every dynamic feature has some runtime cost. I've heard about some proposals to limit the possible
components that the `{{component}}` helper can invoke so the dependency resolution can be done statically.
Whatever solution is designed for the component helper must also be take into consideration for the `{{helper`}} helper.

# Alternatives

The RFC already discusses the alterniative paths since they are the main subject of the proposal.

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?