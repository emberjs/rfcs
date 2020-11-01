---
Start Date: 2019-02-13
Relevant Teams: Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/445
Tracking: https://github.com/emberjs/rfc-tracking/issues/40

---

# Deprecate `{{with}}`

## Summary

The `{{with}}` helper has always had slightly confusing conditional semantics, this was one of the motivators for [introducing](https://github.com/emberjs/rfcs/blob/master/text/0286-block-let-template-helper.md) the easier to understand `{{let}}` helper. Now that `{{let}}` exists, the remaining use case for using `{{with}}` is for its unique conditional semantics. These conditional semantics can be cleanly represented with a combination of `{{let}}` and `{{if}}` statements so we should deprecate `{{with}}`.

## Motivation

The difference between `{{let}}` and `{{with}}` is with how they handle conditional arguments. The `{{let}}` helper's block content is always rendered, regardless of its parameters. In contrast, `{{with}}` only renders its main block when the first position parameter is truthy. For example:

```hbs
{{#with "Alex" as |value|}}
  {{value}} is truthy
{{else}}
  The first positional param was falsy
{{/with}}
```

Will render "[Alex] is truthy".

```hbs
{{#with false as |value|}}
  {{value}} is truthy
{{else}}
  The first positional param was falsy
{{/with}}
```

Will render "The first positional param was falsy".

The conditional arguments behavior of `{{with}}` can easily be replicated using a combination of `{{let}}` and `{{if}}` in a way that's easily readable:

```hbs
{{#let "Alex" as |value|}}
  {{#if value}}
    {{value}} is truthy
  {{else}}
    The first positional param was falsy
  {{/if}}
{{/let}}
```

## Detailed design

We'll create an AST transform in [`packages/ember-template-compiler`](https://github.com/emberjs/ember.js/tree/master/packages/ember-template-compiler) which will emit a deprecation warning for all uses of `{{with}}`. The deprecation warning will be:

```
Using `{{with}}` is deprecated, please use `{{let}}` instead.
```

This message will link to the following deprecation details which aim to give clear guidance on how to migrate to using `{{let}}`, `{{if}}` and `{{else}}` in different combinations:

----

The use of `{{with}}` has been deprecated, you should replace it with either `{{let}}` or a combination of `{{let}}`, `{{if}}` and `{{else}}`:

**If you always want the block to render, replace `{{with}}` with `{{let}}` directly:**

Before:

```hbs
{{#with (hash name="Ben" age=4) as |person|}}
  Hi {{person.name}}, you are {{person.age}} years old.
{{/with}}
```

After:

```hbs
{{#let (hash name="Ben" age=4) as |person|}}
  Hi {{person.name}}, you are {{person.age}} years old.
{{/let}}
```

**If you want to render a block conditionally, use a combination of `{{let}}` and `{{if}}`:**

Before:

```hbs
{{#with user.posts as |blogPosts|}}
  There are {{blogPosts.length}} blog posts
{{/with}}
```

After:

```hbs
{{#let user.posts as |blogPosts|}}
  {{#if blogPosts}}
    There are {{blogPosts.length}} blog posts
  {{/if}}
{{/let}}
```

**If you want to render a block conditionally, and otherwise render an alternative block, use a combination of `{{let}}`, `{{if}}` and `{{else}}`:**

Before:

```hbs
{{#with user.posts as |blogPosts|}}
  There are {{blogPosts.length}} blog posts
{{else}}
  There are no blog posts
{{/with}}
```

After:

```hbs
{{#let user.posts as |blogPosts|}}
  {{#if blogPosts}}
    There are {{blogPosts.length}} blog posts
  {{else}}
    There are no blog posts
  {{/if}}
{{/let}}
```

---

For people on older versions of Ember that support `{{let}}`, we'll create an `ember-template-lint` rule that they can use to prevent the use of `{{with}}`.

We'll also create a codemod which will assist people when migrating from `{{with}}` to `{{let}}`.

## How we teach this

We'll mentiton the deprecation in an Ember point release blog post.

As mentioned above, the deprecation message will contain a link to clear guidelines on how to migrate to using `{{let}}`.

There is nothing to remove from the Ember.js Guides since we already teach only the use of `{{let}}`.
## Drawbacks

This adds a little churn to Ember's API.

## Alternatives

We could leave `{{with}}` as is. I don't believe that this is a good option as the name `{{with}}` is confusing.

We could deprecate `{{with}}` and introduce `{{if-let}}` in Ember core. This RFC originally made that exact proposal, I was strongly persuaded of the [lack of need for `{{if-let}}` by @tcjr](https://github.com/emberjs/rfcs/pull/445#issuecomment-463594185).

We could deprecate `{{with}}` and introduce `{{if-let}}` in an addon instead of Ember core.

## Unresolved questions

(none)
