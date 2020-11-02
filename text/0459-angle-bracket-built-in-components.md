---
Start Date: 2019-03-05
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/459
Tracking: https://github.com/emberjs/rfc-tracking/issues/36

---

# Angle Bracket Invocations For Built-in Components

## Summary

[RFC #311](./0311-angle-bracket-invocation.md) introduced the angle bracket
component invocation syntax. Many developers in the Ember community have since
adopted this feature with very positive feedback. This style of component
invocation will become the default style in the Octane edition and become the
primary way component invocations are taught.

However, Ember ships with three built-in components – `{{link-to}}`, `{{input}}`
and `{{textarea}}`. To date, it is not possible to invoke them with the angle
bracket syntax due to various API mismatches and implementation details.

This RFC proposes some small amendments to these APIs and their implementations
to allow them to be invoked with the angle bracket syntax, i.e. `<LinkTo>`,
`<Input>` and `<Textarea>`.

## Motivation

As mentioned above, this will allow Ember developers to invoke components with
a consistent syntax, which should make it easier to teach.

This RFC _does not_ aim to "fix" issues or quirks with the existing APIs – it
merely attempts to provide a way to do the equivalent invocation in angle
bracket syntax.

## Detailed design

### `<LinkTo>`

There are two main problem with `{{link-to}}`:

* It uses positional arguments as the main API.
* It supports an "inline" form (i.e. without a block).

In the new world, components are expected to work with named arguments. This is
both to improve clarity and to match the HTML tags model (which angle bracket
invocations are loosely modelled after). Positional arguments are reserved for
"control-flow-like" components (e.g. `liquid-if`) and to be paired with the
curly bracket invocation style. Since links are not that, it is not appropiate
for this component to use positional params.

When invoked with a block, the first argument is the route to navigate to. We
propose to name this argument explicitly, with `@route`:

```hbs
{{#link-to "about"}}About Us{{/link-to}}

...becomes...

<LinkTo @route="about">About Us</LinkTo>
```

The second argument can be used to provide a model to the route. We propose to
name this argument explicitly, with `@model`:

```hbs
{{#let this.model.posts.firstObject as |post|}}

  {{#link-to "post" post}}Read {{post.title}}...{{/link-to}}

  ...becomes...

  <LinkTo @route="post" @model={{post}}>Read {{post.title}}...</LinkTo>

{{/let}}
```

In fact, it is possible to pass multiple models to deeply nested routes with
additional positional arguments. For this use case, we propose the `@models`
named argument which accepts an array:

```hbs
{{#let this.model.posts.firstObject as |post|}}
  {{#each post.comments as |comment|}}

    {{#link-to "post.comment" post comment}}
      Comment by {{comment.author.name}} on {{comment.date}}
    {{/link-to}}

    ...becomes...

    <LinkTo @route="post.comment" @models={{array post comment}}>
      Comment by {{comment.author.name}} on {{comment.date}}
    </LinkTo>

  {{/each}}
{{/let}}
```

The singular `@model` argument is a special case of `@models`, provided as a
convenience for the common case. Passing both `@model` and `@models` will be an
error. Passing insufficient amount of models for the given route, will continue
to be an error.

It is also possible to pass query params to the `{{link-to}}` component with
the somewhat awkward `(query-params)` API. We propose to replace it with a
`@query` named argument that simply take a regular hash (or POJO):

```hbs
{{#link-to "posts" (query-params direction="desc" showArchived=false)}}
  Recent Posts
{{/link-to}}

...becomes...

<LinkTo @route="posts" @query={{hash direction="desc" showArchived=false}}>
  Recent Posts
</LinkTo>
```

Finally, as mentioned above, `{{link-to}}` supports an "inline" form without a
block. This form doesn't bring much value and causes confusion around
the ordering of the arguments. We propose to simply not support this for the
angle bracket invocation style:

```hbs
{{link-to "About Us" "about"}}

...becomes...

<LinkTo @route="about">About Us</LinkTo>
```

Other APIs of this compoment are already based on named arguments.

#### Migration Path

We would provide a codemod to convert the old invocation style into the new
style.

#### Template Lints

Even though the angle bracket invocation style is recommended going forward,
components can generally be invoked using the either the curly or angle bracket
syntax. Therefore, while not recommended, `{{link-to}}` would still work and
invoke the same component.

We propose to add template lint rules to using this component with the curly
invocation style.

### `<Input>`

Today, the `{{input}}` component is internally implemented as several internal
components that are selected based on the `type` argument. This is intended as
an internal implementation detail, but as a result, it is not possible to invoke
the component with `<Input>` since it does not exist as a "real" component.

We propose to change this internal implementation strategy to make it possible
to invoke it with angle brackets just like any other components.

For example:

```hbs
{{input type="text" value=this.model.name}}

...becomes...

<Input @type="text" @value={{this.model.name}} />
```

Another example:

```hbs
{{input type="checkbox" name="email-opt-in" checked=this.model.emailPreference}}

...becomes...

<Input @type="checkbox" @name="email-opt-in" @checked={{this.model.emailPreference}} />
```

#### Migration Path

We would provide a codemod to convert the old invocation style into the new
style.

#### Template Lints

Even though the angle bracket invocation style is recommended going forward,
components can generally be invoked using the either the curly or angle bracket
syntax. Therefore, while not recommended, `{{input}}` would still work and
invoke the same component.

We propose to add template lint rules to using this component with the curly
invocation style.

### `<Textarea>`

Due to a similar implementation issue, it is also not possible to invoke the
`{{textarea}}` component with angle bracket invocation style.

We propose to change this internal implementation strategy to make it possible
to invoke this with angle brackets just like any other components.

For example:

```hbs
{{textarea value=this.model.body}}

...becomes...

<Textarea @value={{this.model.body}} />
```

#### Migration Path

We would provide a codemod to convert the old invocation style into the new
style.

[RFC #176](./0176-javascript-module-api.md) picked `text-area`/`TextArea` for
this component. To prevent confusion, we will add a helpful hint to the error
message ("did you mean `<Textarea>`?") when a user mistakenly typed
`{{text-area}}` or `<TextArea>` in development mode.

#### Template Lints

Even though the angle bracket invocation style is recommended going forward,
components can generally be invoked using the either the curly or angle bracket
syntax. Therefore, while not recommended, `{{textarea}}` would still work and
invoke the same component.

We propose to add template lint rules to using this component with the curly
invocation style.

## How we teach this

Going forward, we will focus on teaching the angle bracket invocation style as
the main (only?) way of invoking components. In that world, there wouldn't be
anything extra to teach, as the invocation style proposed in this RFC is not
different from any other components, which is the purpose of this proposal. Of
course, the APIs of these components will still need to be taught, but that is
not a new change.

The only caveat is that, since the advanced `<LinkTo>` APIs require passing
arrays and hashes, the `{{array}}` and `{{hash}}` helper would have to be
taught before those advanced features can be introduced. However, since the
basic usage (linking to top-level routes) does not require either of those
helpers, it doesn't really affect things from a getting started perspective.

It should also be mentioned that, other built-ins, such as `{{yield}}`,
`{{outlet}}`, `{{mount}}`, etc are considered "keywords" not components, they
are also "control-flow-like", so it wouldn't be appropiate to invoke them with
angle brackets.

The technical implementation of this RFC will need to be accompanied by changes to the API docs for the built-in template helpers, [link-to](https://emberjs.com/api/ember/3.8/classes/Ember.Templates.helpers/methods/link-to?anchor=link-to) and [input](https://emberjs.com/api/ember/3.8/classes/Ember.Templates.helpers/methods/input?anchor=input). In keeping with past decisions for Angle Brackets, the API docs should show both curly and Angle Bracket invocations of these helpers. The API docs are expected to show the full supported API surface of Ember.

`link-to` and `input` are used liberally throughout the Ember.js Guides, Tutorial, and `super-rentals` [sample app](https://github.com/ember-learn/super-rentals), so those examples will need to be updated. In the Guides, we want to show solely Angle Brackets invocation. The [syntax conversion guide](https://guides.emberjs.com/release/reference/syntax-conversion-guide/) should be revised to include these helpers.
## Drawbacks

None.

## Alternatives

* `<LinkTo>` could only support `@models` without special casing `@model` as a
  convenience.

* `<LinkTo>` could support a `@text` argument for inline usage.

* `<Textarea>` could be named `<TextArea>`.

## Unresolved questions

None.
