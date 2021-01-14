---
Stage: Accepted
Start Date: 2020-12-22
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/689
---

# Deprecate `{{hasBlock}}` and `{{hasBlockParams}}` in templates

## Summary

Deprecate the `{{hasBlock}}` and `{{hasBlockParams}}` properties in templates in
favor of the `{{has-block}}` and `{{has-block-params}}` keywords.

## Motivation

`{{hasBlock}}` is a way to check if component has a default block provided to
it, and `{{hasBlockParams}}` is a way to check if that default block has block
parameters. They are effectively aliases for calling `{{has-block}}` and
`{{has-block-params}}` respectively, without providing a block name or with the
block name `"default"`. They are also not called, and acts like a template
fallback lookup instead.

```hbs
{{! hasBlock and hasBlockParams can be referenced directly }}
{{#if hasBlock}}{{/if}}

{{#if hasBlockParams}}{{/if}}


{{! has-block and has-block-params must be called }}
{{#if (has-block)}}{{/if}}

{{#if (has-block-params)}}{{/if}}
```

Having two ways of accomplishing this task is confusing, and with the property
form it is not possible to check if other blocks exist, such as named blocks. As
such, this RFC proposes that we deprecate `{{hasBlock}}` and
`{{hasBlockParams}}` and recommend that users switch to using `{{has-block}}`
and `{{has-block-params}}`.

## Transition Path

Users who are currently using `{{hasBlock}}` or `{{hasBlockParams}}` will need
to replace all of their usages with `{{has-block}}`/`{{has-block-params}}`
respectively. This is generally a very codemoddable change, so it should be
pretty straightforward to accomplish, and we will attempt to make a codemod to
help out with the transition.

## How We Teach This

### Deprecation Guide

#### `{{hasBlock}}`

The `{{hasBlock}}` property is true if the component was given a default block,
and false otherwise. To transition away from it, you can use the `(has-block)`
helper instead.

```hbs
{{hasBlock}}

{{! becomes }}
{{has-block}}
```

Unlike `{{hasBlock}}`, the `(has-block)` helper must be called, so in nested
positions you will need to add parentheses around it:

```hbs
{{#if hasBlock}}

{{/if}}


{{! becomes }}
{{#if (has-block)}}

{{/if}}
```

You may optionally pass a name to `(has-block)`, the name of the block to check.
The name corresponding to the block that `{{hasBlock}}` represents is "default".
Calling `(has-block)` without any arguments is equivalent to calling
`(has-block "default")`.

#### `{{hasBlockParams}}`

The `{{hasBlockParams}}` property is true if the component was given a default block,
and false otherwise. To transition away from it, you can use the `(has-block-params)`
helper instead.

```hbs
{{hasBlockParams}}

{{! becomes }}
{{has-block-params}}
```

Unlike `{{hasBlockParams}}`, the `(has-block-params)` helper must be called, so in nested
positions you will need to add parentheses around it:

```hbs
{{#if hasBlockParams}}

{{/if}}


{{! becomes }}
{{#if (has-block-params)}}

{{/if}}
```

You may optionally pass a name to `(has-block-params)`, the name of the block to check.
The name corresponding to the block that `{{hasBlockParams}}` represents is "default".
Calling `(has-block-params)` without any arguments is equivalent to calling
`(has-block-params "default")`.

## Drawbacks

- Introduces some churn.

- `(has-block)` is dasherized, which is not inline with the future of strict
  mode and template imports. Dasherized strings are not valid identifiers in
  JavaScript, so helpers and modifiers will likely switch to camel case when
  that transition occurs.

  This actually makes this deprecation even more valuable. As it stands, even if
  we wanted to use `{{hasBlock "name"}}` in templates, we could not, since
  `{{hasBlock}}` already has semantics and is not callable. By deprecating this
  syntax, we can reclaim it in the future as a possible alias for the dasherized
  version.

## Alternatives

- Keep the existing syntax as an alias for `(has-block)`/`(has-block-params)`.
