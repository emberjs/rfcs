---
Stage: Released
Start Date: 2021-01-05
Release Date: 2021-03-22
Release Versions:
  ember-source: v3.26.0
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/698
---

<!---
Directions for above:

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Deprecate <LinkTo> Component Positional Arguments

## Summary

We propose to deprecate invoking the `<LinkTo>` component with positional
arguments, in favor of the equivalent named arguments introduced in
[RFC #459](0459-angle-bracket-built-in-components.md). We also propose to
deprecate the `(query-params)` helper, which is only needed when invoking the
`<LinkTo>` component with positional arguments.

## Motivation

In modern Ember, the idiomatic way to invoke most components is to use the
[angle bracket syntax](0311-angle-bracket-invocation.md) along with the named
arguments `@` syntax. On the other hand, curly invocations are now reserved for
"helper-like" and "control-flow" components.

The `<LinkTo>` built-in component started life as a "helper" in the early days
of Ember, even before an official components API was created. Over time, it
became clear that `<LinkTo>` is better classified as a component than a helper
in modern Ember, and the learning materials have been updated accordingly.

This historical origin meant that `<LinkTo>` was designed to accept positional
arguments exclusively, as was common with most helpers at the time. The angle
bracket syntax, on the other hand, accepts named arguments exclusively.

Another downside to the positional arguments syntax was it required the use of
the `(query-params)` helper to distinguish query params from model arguments.

Finally, the meaning of the positional arguments also changes slightly when the
component is invoked with or without a block, which made things needlessly
confusing.

To address these issues, [RFC #459](0459-angle-bracket-built-in-components.md)
introduced explicit equivalent names for the positional arguments that were
accepted by the `<LinkTo>` component. This allowed it to be invoked with the
same angle bracket syntax and avoided the other confusions mentioned above.

These features were made available since v3.10 and are the idomatic thing to do
in modern Ember codebase.

Given that the feature are now available on all currently-supported Ember
versions and the community had adequate time to make the transition, this would
be a good time to deprecate the obsolete features to reduce confusion as well
as implementation complexity.

In particular, some of the obsolete features required capabilities not usually
available to other components, such as knowing whether a block was passed or
not and relies on "AST transforms" to normalize some of the differences. These
implementation strategies introduces unnecessary complexity in the internals
that sometimes causes bugs or other surprising behaviors.

## Transition Path

```hbs
Deprecated:

{{link-to "About Us" "about"}}
          ~~~~~~~~~~~~~~~~~~

Invoking the `<LinkTo>` component with positional arguments is deprecated.
Instead, please use the equivalent named arguments (`@route`) and pass a
block for the link's content.

<LinkTo @route="about">About Us</LinkTo>
```

```hbs
Deprecated:

{{#link-to "about"}}About Us{{/link-to}}
           ~~~~~~~

Invoking the `<LinkTo>` component with positional arguments is deprecated.
Instead, please use the equivalent named arguments (`@route`).

Replacement:

<LinkTo @route="about">About Us</LinkTo>
```


```hbs
Deprecated:

{{#link-to "post" @post}}Read {{@post.title}}...{{/link-to}}
           ~~~~~~~~~~~~

Invoking the `<LinkTo>` component with positional arguments is deprecated.
Instead, please use the equivalent named arguments (`@route`, `@model`).

Replacement:

<LinkTo @route="post" @model={{@post}}>Read {{@post.title}}...</LinkTo>
```

```hbs
Deprecated:

{{#link-to "post.comment" @comment.post @comment}}
           ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  Comment by {{@comment.author.name}} on {{@comment.date}}
{{/link-to}}

Invoking the `<LinkTo>` component with positional arguments is deprecated.
Instead, please use the equivalent named arguments (`@route`, `@models`).

Replacement:

<LinkTo @route="post.comment" @models={{array post comment}}>
  Comment by {{comment.author.name}} on {{comment.date}}
</LinkTo>
```

```hbs
Deprecated:

{{#link-to "posts" (query-params direction="desc" showArchived=false)}}
           ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
  Recent Posts
{{/link-to}}

Invoking the `<LinkTo>` component with positional arguments is deprecated.
Instead, please use the equivalent named arguments (`@route`, `@query`) and the
`hash` helper.

Replacement:

<LinkTo @route="posts" @query={{hash direction="desc" showArchived=false}}>
  Recent Posts
</LinkTo>
```

These migrations can be automated using the [angle brackets codemod](https://github.com/ember-codemods/ember-angle-brackets-codemod).

## How We Teach This

The Octane learning materials have already been updated to use the latest
idioms, so no changes are necessary. The API documentation should be updated
to mark the obsolete features and the `(query-params)` helper as deprecated.

A deprecation guide will need to be written using the same examples from the
[Transition Path](#transition-path) section. The guide should also promote the
use of the codemod to automate the migration.

## Drawbacks

None.

## Alternatives

None.

## Unresolved questions

None.
