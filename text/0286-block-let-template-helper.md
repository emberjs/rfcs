---
Start Date: 2017-12-21
RFC PR: https://github.com/emberjs/rfcs/pull/286
Ember Issue: https://github.com/emberjs/ember.js/pull/16076

---

# Block `let` template helper

## Summary

Introduce the `let` template helper in block form.

## Motivation

The goal of this RFC is to introduce a `let` template helper that allows to create new bindings in templates.
The design of this helper is similar to `with`,
but without the conditional rendering of the block depending on the values passed into the helper.

While the conditional semantics of `with` are coherent with the other built-in helpers like `each` and `if`,
users often find this unexpected.
The fact that only the first positional parameter of `with` controls whether the block is rendered might also add to the confusion.

Taking an example from [RFC #200](https://github.com/emberjs/rfcs/pull/200),
let's consider we have the following template:

```handlebars
Welcome back {{concat (capitalize person.firstName) ' ' (capitalize person.lastName)}}

Account Details:
First Name: {{capitalize person.firstName}}
Last Name: {{capitalize person.lastName}}
```

Because you have to know to capitalize every time you want to display a name,
errors might be introduced if we forget to do it when adding the name somewhere else in the template.
Using the `let` helper, this could be done like so:

```handlebars
{{#let (capitalize person.firstName) (capitalize person.lastName)
  as |firstName lastName|
}}
  Welcome back {{concat firstName ' ' lastName}}

  Account Details:
  First Name: {{firstName}}
  Last Name: {{lastName}}
{{/let}}
```

Now you can use `firstName` and `lastName` inside the `let` block with the knowledge that that logic is in a single place.

With the introduction of template-only components in [RFC #278](https://github.com/emberjs/rfcs/pull/278),
having the capability to create additional bindings in the template would prove useful.
Another aspect to consider is related to the [Named Blocks RFC](https://github.com/emberjs/rfcs/blob/master/text/0226-named-blocks.md).
In both the case of named blocks and block let, you can achieve most of the same functionality by using components.
The components approach has its own drawbacks, which are explored in Alternatives below.

## Detailed design

The `let` helper should be implemented as a built-in helper, with the following semantics:

* **Only** the block form is available
* The block is always rendered
* It should support however many positional arguments are passed to the helper
* Positional arguments passed to the helper should be yielded back out in the same order
* Inline form issues an error, linking users to documentation

There already exists [an implementation in the codebase](https://github.com/emberjs/ember.js/blob/9536e137b9e1a39411b7fd4e8ca0e7fbb341ef17/packages/ember-glimmer/tests/integration/syntax/experimental-syntax-test.js#L6-L37) that can be used as a basis.

## How We Teach This

The introduction of the `let` helper brings no new concepts.
It touches on the concepts of block helpers, how to pass arguments to them,
and how to use block parameters (`as |foo|`), which should already be introduced in the literature.

Current Ember developers should find it familiar to use `let`, as it is very similar to `with`.

JavaScript developers should also be familiar with `let` bindings,
as recent specifications of the language introduced that keyword.

The Guides already possess a section dedicated to Templates, with multiple mentions of helpers.
`let` would likely be documented in the [Built-in Helpers](https://guides.emberjs.com/v2.17.0/templates/built-in-helpers/) guide alongside the others.

If this RFC is approved, the `let` will initially only support the block form.
This means that only the following form is available for users:

```handlebars
{{#let 1 2 3 as |one two three|}}
  A, B, C, easy as {{one}}, {{two}}, {{three}}
{{/let}}
```

This could also be enforced by issuing a helpful error when `let` is used in the inline form.

## Drawbacks

As is the case when adding any sort of API, we will be increasing the cognitive load of learners and users,
as it is one more piece of information to obtain and retain.

The cost of learning this API is mitigated by the fact that its effects are very localized.
It is a template helper, so it will only affect templates.
It is not required for general usage of Ember, unlike something like `link-to`,
so you can learn the helper at your own pace.

And lastly, if you do use it or encounter it in code, only the markup inside the `{{#let}}{{/let}}` block is affected,
making it easier to reason about.

## Alternatives

### Inline form

At the moment, the only way to introduce a new binding in a template is through block params.
For example, if you are iterating over an array with `each`, you
introduce a binding named `item` for the item currently being iterated:

```handlebars
{{#each myArray as |item|}}
  I am item {{item}}.
{{/each}}
```

The inline form of `let` would be an additional way of introducing bindings in templates.
Using the names example from the RFC, it would look like the following in inline form:

```handlebars
{{let
  firstName=(capitalize person.firstName)
  lastName=(capitalize person.lastName)
}}

Welcome back {{concat firstName ' ' lastName}}

Account Details:
First Name: {{firstName}}
Last Name: {{lastName}}
```

This syntax raises questions about the semantics of the inline form,
such as what is the scope of the binding, that are better left to a subsequent RFC.

### Using components

In a similar situation to [Named Blocks RFC](https://github.com/emberjs/rfcs/blob/master/text/0226-named-blocks.md),
it is also possible to replicate some of the behavior of the proposed `let` helper using components.
However, using components also presents some drawbacks.

You can extract the template and do:

```handlebars
// app/templates/components/person-tile.hbs
Welcome back {{concat firstName ' ' lastName)}}

Account Details:
First Name: {{firstName}}
Last Name: {{lastName}}
```

```handlebars
{{person-tile firstName=(capitalize person.firstName) lastName=(capitalize person.lastName)}}
```

This addresses not having to repeat `capitalize` wherever the names are used,
but splits the content into multiple files for the sake of it.
While [module unification](https://github.com/emberjs/rfcs/blob/master/text/0143-module-unification.md) mitigates the locality problem by putting related files in the same folder,
there is still the overhead of having to consult multiple files.

You can instead use a block version of the component as a wrapper to the content.
Some variations are possible: you can pass data into the component as either positional or named arguments;
you can export either an object with the arguments as keys, or export multiple block parameters.

Passing positional arguments to components is onerous,
and necessitates having a JavaScript file to define which positional arguments it accepts.

Passing named arguments to components would be the closest to `let`,
but it would still require a componente template file which would yield them as block parameters.

Yielding out the values is where it gets tricky in components,
regardless of returning a hash or multiple block parameters,
due to the lack of a "splat" operator in Handlebars.

Since you cannot do something like this at the moment:

```handlebars
// app/templates/components/person-tile.hbs
{{yield ...arguments}}
```

You would have to explicitly encode all of the arguments:

```handlebars
// app/templates/components/person-tile.hbs
{{yield firstName lastName}}
```

Or

```handlebars
// app/templates/components/person-tile.hbs
{{yield args=(hash firstName=firstName lastName=lastName)}}
```

Leading to some repetition of names.

This makes the solution of using components brittle to changes,
as typos or ordering mistakes can introduce silent errors in your application.

### Adding named arguments to `with`

[RFC #202](https://github.com/emberjs/rfcs/pull/202) proposes to add named arguments to `with`.

I feel it is less practical to add a new mode to the helper where it always renders,
when its semantics are already confusing to users.
The RFC #202 proposal also presents the problem of bringing back context-switching helpers,
as it proposes omitting block arguments (`as |bar|` in `{{#with foo as |bar|}}`).

### Remove the conditional behavior of `with`

Making the `with` helper unconditionally render the block would be a major breaking change of its semantics,
and would likely affect existing applications in insidious ways.
For this reason, I reject this alternative out of the gate.

### Support `let` via the `ember-let` addon

There is an [`ember-let`](https://github.com/thefrontside/ember-let) addon which implements both the block and the inline forms of `let`.
To implement the necessary functionality, the addon had to resort to private API usage, which is brittle and subject to breakage.

Having `let` available from Ember itself would make sure that it would not be subject to breakage the same way,
and the end user would not have to worry about version compatibility.

## Unresolved questions

None.

## Future work

### Deprecating `with`

With the introduction of the `let` helper, `with` should likely be deprecated.

### `if-let`, `let*` and others

RFC #200 also proposes the `if-let` and `let*` helpers.

`if-let` mimics the behaviour of `with`,
enabling the user to introduce bindings and conditionally rendering the block.
The advantage of introducing `if-let` over using `with` would be to define its semantics without worrying about making breaking changes to `with`.

`let*` would allow bindings to happen sequentially, that is,
`let` (`{{let* a=1 b=(sum a 5)}}` would be valid instead of throwing an error about `a` in `(sum a 5)`.

These could also be addressed in subsequent RFCs, focused on the specificities of each proposal.
