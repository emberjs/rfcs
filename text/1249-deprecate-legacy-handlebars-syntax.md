---
stage: ready-for-release
start-date: 2026-10-14T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - learning
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1249'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1254'
project-link:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
-->

# Deprecate legacy Handlebars template syntax

## Summary

Deprecate four pieces of template syntax that Ember inherited from the Handlebars parser, each of
which has a documented equivalent: the `{{&foo}}` mustache, `{{^}}` as the
start of an else section, blocks that start with `{{^`, and quadruple-curly raw blocks
(`{{{{foo}}}}...{{{{/foo}}}}`). 

## Motivation

Ember's template parser is built on the Handlebars parser, and it accepts some Handlebars syntax
that Ember has never documented or taught, and that nothing in Ember or Glimmer tested. Each one is a second spelling of something Ember already has, and two of
them behave in ways that surprise the people who use them:

- **`{{&foo}}`** is an unescaped mustache, exactly like `{{{foo}}}`.
- **`{{^}}`** starts the else section of a block, exactly like `{{else}}`.
- **`{{^foo}}a{{else}}b{{/foo}}`** swaps the two halves of the block: `a` renders when `foo` would
  render its _inverse_, and `b` when it would render its main block. It reads as the opposite of
  what it does. Block parameters on such a block are silently ignored:
  `{{^each this.items as |item|}}...{{else}}{{item}}{{/each}}` never binds `item`, so `{{item}}` is
  looked up as an unrelated name. (Without an `{{else}}`, `{{^foo}}...{{/foo}}` is already a syntax
  error.)
- **Raw blocks** (`{{{{foo}}}}...{{{{/foo}}}}`) are only half raw. The body is not processed for
  mustaches, but it is still parsed as HTML, so `{{{{foo}}}}<div>{{{{/foo}}}}` is an "Unclosed
  element" error. The same result is available with an ordinary block and `\{{` escapes.

Every tool that reads Ember templates has to support these forms, since a template that uses them
compiles today: Prettier, ember-template-lint, Glint and the language servers, codemods, and any
alternative implementation of the template compiler. Deprecating and then removing them shrinks
the template syntax to what Ember documents, with no loss of expressiveness: each form has a
direct, mechanical replacement, and Prettier already applies most of them (see "Transition Path").

## Transition Path

Each form gets its own deprecation, with `until: '8.0.0'`, enabled in the release that ships it.
The deprecations are reported by the template compiler when it compiles a template, with the
module name and the line and column of the construct, the way earlier template-syntax
deprecations were (for example `ember-template-compiler.deprecate-render`). The parse tree cannot
tell these forms from their replacements (`{{&foo}}` and `{{{foo}}}` produce the same node), so the
parser detects them from the source text while parsing.

| Id                                 | Deprecated                   | Replacement                                                 | Prettier                              |
| ---------------------------------- | ---------------------------- | ----------------------------------------------------------- | ------------------------------------- |
| `template-ampersand-mustache`      | `{{&foo}}`                   | `{{{foo}}}`                                                 | fixes it, except with `~` (see below) |
| `template-caret-else`              | `{{^}}`                      | `{{else}}`                                                  | fixes it                              |
| `template-inverse-block-with-else` | `{{^foo}}a{{else}}b{{/foo}}` | `{{#foo}}b{{else}}a{{/foo}}`                                | fixes it                              |
| `template-raw-block`               | `{{{{foo}}}}...{{{{/foo}}}}` | `{{#foo}}...{{/foo}}`, with `\{{` for each `{{` in the body | fixes it                              |

The replacements produce exactly the same template, including whitespace control: in every case
the `~` stays on the delimiter it was on, except for `{{&foo}}`, where it moves between the braces
(`{{~&foo~}}` becomes `{{~{foo}~}}`). This was checked by parsing each pair.

For `{{^foo}}...{{else}}...{{/foo}}` with block parameters, the replacement that keeps today's
behavior drops the block parameters, since they never worked. The deprecation guide explains this
and shows the form the author almost certainly meant, with the parameters on the `{{#foo}}`.

**Prettier.** Prettier's `glimmer` printer, and `prettier-plugin-ember-template-tag` for `.gjs`
and `.gts` files, already print all four replacements, because they print from the parse tree.
Running Prettier is therefore the fastest fix for most apps. One case is wrong: a `{{&foo}}` with
whitespace control prints as `{{{~foo}}}` or `{{{foo~}}}`, which is a syntax error. The
`template-ampersand-mustache` guide tells people to fix those by hand, and the Prettier printer
should be fixed to print `{{~{foo}}}` and `{{{foo}~}}`. This was checked with Prettier 2.8.8,
3.0.0, 3.8.3 and 3.9.10.

**Ecosystem.**

- _Addons._ An addon whose published templates use these forms triggers the deprecation in the
  apps that compile them (v1 addons) or in its own build (v2 addons). The addon has to be updated
  before Ember 8.0; app authors who see the deprecation for an addon's template should report it
  to the addon.
- _IDE support, Glint, codemods._ Tools that parse templates with `@glimmer/syntax` keep working
  through the deprecation period; after removal, the forms can become syntax errors there too.
- _Ember Inspector, server-side rendering, Ember Engines, blueprints._ Not affected: this is
  compile-time syntax only, and no blueprint uses it.

## How We Teach This

None of these forms appears in the Ember guides or API docs, so nothing there changes. The
deprecation guides below are the teaching material. They are written for someone who has just
seen the deprecation message and wants to fix their templates: each one says what the form means,
whether Prettier fixes it, how to find every occurrence, and how to fix it by hand.

<details><summary>Deprecation guide <code>template-ampersand-mustache</code>: Unescaped <code>{{&foo}}</code> mustaches</summary>

The `{{&foo}}` form of an unescaped mustache is deprecated. It comes from Handlebars and means
exactly the same as the triple-curly form `{{{foo}}}`, which is the only form Ember documents.
Replace `{{&` with `{{{` and the closing `}}` with `}}}` (see below if the mustache uses `~`).

### Fixing it automatically with Prettier

If your project formats templates with [Prettier](https://prettier.io/), running it fixes this
deprecation for you: Prettier prints `{{&foo}}` as `{{{foo}}}`. This works for `.hbs` files and,
with
[`prettier-plugin-ember-template-tag`](https://github.com/ember-tooling/prettier-plugin-ember-template-tag),
for `<template>` tags in `.gjs` and `.gts` files.

```bash
npx prettier --write "app/**/*.{hbs,gjs,gts}"
```

Prettier also reformats anything else in those files that is not already formatted. If you only
want this change, use the manual fix below.

**Except with whitespace control.** If the mustache has a `~`, as in `{{~&foo}}` or `{{&foo~}}`,
Prettier prints `{{{~foo}}}` or `{{{foo~}}}`, which is a syntax error. Fix those by hand, as
shown below. (Checked with Prettier 2.8 through 3.9.)

### Fixing it by hand

To find every occurrence, search your templates for `{{&` (including the `{{~&` form):

```bash
grep -rnE '\{\{~?&' app
```

Before:

```handlebars
<p>{{{this.renderedMarkdown}}}</p>
<div title={{{@title}}}></div>
{{{~this.footer~}}}
```

After:

```handlebars
<p>{{{this.renderedMarkdown}}}</p>
<div title={{{@title}}}></div>
{{{~this.footer~}}}
```

Arguments and named arguments stay as they are: `{{&format-html @value mode="short"}}` becomes
`{{{format-html @value mode="short"}}}`. When you use whitespace control, each tilde goes between
the first two or the last two braces: `{{~&foo~}}` becomes `{{~{foo}~}}`, `{{~&foo}}` becomes
`{{~{foo}}}`, and `{{&foo~}}` becomes `{{{foo}~}}`.

Both forms insert the value into the page as HTML without escaping it. This change keeps that
behavior exactly. If the value can contain user input, consider whether it should be unescaped at
all.

</details>

<details><summary>Deprecation guide <code>template-caret-else</code>: Using <code>{{^}}</code> instead of <code>{{else}}</code></summary>

Using `{{^}}` to start the `{{else}}` section of a block is deprecated. It comes from Handlebars
and means exactly the same as `{{else}}`. Replace each `{{^}}` with `{{else}}`.

### Fixing it automatically with Prettier

If your project formats templates with [Prettier](https://prettier.io/), running it fixes this
deprecation for you: Prettier always prints `{{^}}` as `{{else}}`, and keeps any `~` whitespace
control. This works for `.hbs` files and, with
[`prettier-plugin-ember-template-tag`](https://github.com/ember-tooling/prettier-plugin-ember-template-tag),
for `<template>` tags in `.gjs` and `.gts` files.

```bash
npx prettier --write "app/**/*.{hbs,gjs,gts}"
```

Prettier also reformats anything else in those files that is not already formatted. If you only
want this change, use the manual fix below.

### Fixing it by hand

To find every occurrence, search your templates for `{{^}}` (including the `{{~^~}}` forms):

```bash
grep -rnE '\{\{~?\^[[:space:]]*~?\}\}' app
```

Before:

```handlebars
{{#if this.isLoggedIn}}
  Welcome back!
{{else}}
  Please log in.
{{/if}}

{{#each this.items as |item|}}
  <li>{{item.name}}</li>
{{~else~}}
  <li>No items</li>
{{/each}}
```

After:

```handlebars
{{#if this.isLoggedIn}}
  Welcome back!
{{else}}
  Please log in.
{{/if}}

{{#each this.items as |item|}}
  <li>{{item.name}}</li>
{{~else~}}
  <li>No items</li>
{{/each}}
```

Keep any `~` where it was: `{{~^}}` becomes `{{~else}}`, `{{^~}}` becomes `{{else~}}`, and
`{{~^~}}` becomes `{{~else~}}`.

This deprecation is only about `{{^}}` on its own. A block that _starts_ with `{{^`, such as
`{{^if this.isLoggedIn}}`, is a different deprecated form with
[its own guide](/deprecations/v7.x/#toc_template-inverse-block-with-else).

</details>

<details><summary>Deprecation guide <code>template-inverse-block-with-else</code>: Blocks that start with <code>{{^</code></summary>

Starting a block with `{{^` instead of `{{#`, as in `{{^if this.isLoggedIn}}...{{else}}...{{/if}}`,
is deprecated. This "inverse block" form comes from Handlebars. It swaps the two halves of the
block: the part before `{{else}}` is what renders when the condition is _false_, and the part after
`{{else}}` is what renders when it is _true_.

To fix it, start the block with `{{#` and swap the two halves, so that it reads the way it
renders. (An inverse block _without_ `{{else}}`, like `{{^if x}}...{{/if}}`, is already a syntax
error.)

### Fixing it automatically with Prettier

If your project formats templates with [Prettier](https://prettier.io/), running it fixes this
deprecation for you: Prettier prints `{{^foo}}a{{else}}b{{/foo}}` as `{{#foo}}b{{else}}a{{/foo}}`,
which renders the same thing. This works for `.hbs` files and, with
[`prettier-plugin-ember-template-tag`](https://github.com/ember-tooling/prettier-plugin-ember-template-tag),
for `<template>` tags in `.gjs` and `.gts` files.

```bash
npx prettier --write "app/**/*.{hbs,gjs,gts}"
```

Prettier also reformats anything else in those files that is not already formatted. If you only
want this change, use the manual fix below. And if the block has block parameters (`as |item|`),
read "Block parameters" below first.

### Fixing it by hand

To find every occurrence, search your templates for a block that starts with `{{^` followed by a
name:

```bash
grep -rnE '\{\{~?\^[[:space:]]*[^}~[:space:]]' app
```

Before:

```handlebars
{{#if this.isLoggedIn}}
  Welcome back!
{{else}}
  Please log in.
{{/if}}
```

After, with the two halves swapped:

```handlebars
{{#if this.isLoggedIn}}
  Welcome back!
{{else}}
  Please log in.
{{/if}}
```

For `if`, you can instead keep the halves in their original order and use `unless`, which also
renders the same thing:

```handlebars
{{#unless this.isLoggedIn}}
  Please log in.
{{else}}
  Welcome back!
{{/unless}}
```

The same swap works for any block, for example `each`:

Before:

```handlebars
{{#each this.results}}
  {{this.results.length}}
  results found.
{{else}}
  No results.
{{/each}}
```

After:

```handlebars
{{#each this.results}}
  {{this.results.length}}
  results found.
{{else}}
  No results.
{{/each}}
```

If you use `~` whitespace control, keep each tilde on the same delimiter it was on: the opening
`{{^each ...~}}` becomes `{{#each ...~}}`, and the tildes on `{{else}}` and `{{/each}}` stay where
they are.

### Block parameters

Block parameters on an inverse block never worked. In
`{{^each this.items as |item|}}...{{else}}{{item}}{{/each}}`, the `as |item|` is ignored, so the
`{{item}}` after `{{else}}` does not refer to the current item: it is looked up as a separate name.
Prettier's fix keeps that behavior and drops the `as |item|`.

If you meant the block parameter to work, which is almost certainly the case, write the block the
way it was intended, with `as |item|` on the `{{#each}}`:

Before:

```handlebars
{{#each this.items}}
  <li>{{item.name}}</li>
{{else}}
  No items.
{{/each}}
```

After:

```handlebars
{{#each this.items as |item|}}
  <li>{{item.name}}</li>
{{else}}
  No items.
{{/each}}
```

This renders the items' names, which the original template did not, so check the result.

</details>

<details><summary>Deprecation guide <code>template-raw-block</code>: Raw blocks (<code>{{{{foo}}}}...{{{{/foo}}}}</code>)</summary>

Raw blocks, written with four curly braces as in `{{{{my-component}}}}...{{{{/my-component}}}}`,
are deprecated. They come from Handlebars. A raw block is an ordinary block invocation of
`my-component` whose body is not processed for mustaches: any `{{...}}` inside it is literal text.
HTML inside the body is still parsed as HTML.

To fix it, write the block with `{{#` and `{{/`, and escape each mustache in its body with a
backslash, `\{{`, so that it stays literal text.

### Fixing it automatically with Prettier

If your project formats templates with [Prettier](https://prettier.io/), running it fixes this
deprecation for you: Prettier prints a raw block as an ordinary block with every `{{` in its body
escaped as `\{{`, which renders the same thing. This works for `.hbs` files and, with
[`prettier-plugin-ember-template-tag`](https://github.com/ember-tooling/prettier-plugin-ember-template-tag),
for `<template>` tags in `.gjs` and `.gts` files.

```bash
npx prettier --write "app/**/*.{hbs,gjs,gts}"
```

Prettier also reformats anything else in those files that is not already formatted. If you only
want this change, use the manual fix below.

### Fixing it by hand

To find every occurrence, search your templates for four opening braces:

```bash
grep -rn '{{{{' app
```

Then, for each raw block:

1. Change the opening `{{{{name ...}}}}` to `{{#name ...}}`, keeping any arguments.
2. Change the closing `{{{{/name}}}}` to `{{/name}}`.
3. In the body, put a backslash in front of every `{{`, so it stays literal text.

Before:

```handlebars
{{#code-sample language="handlebars"}}
  <p>Hello, \{{@name}}!</p>
{{/code-sample}}
```

After:

```handlebars
{{#code-sample language="handlebars"}}
  <p>Hello, \{{@name}}!</p>
{{/code-sample}}
```

Both render the `code-sample` component with the text `Hello, {{@name}}!` inside a `<p>` in its
block. Only `{{` needs the backslash: `}}` is ordinary text. A triple curly in the body is escaped
the same way, as `\{{{`.

If the body was in a raw block so that it could show template code, you may find it clearer to
pass that code as a string instead, for example
`<CodeSample @language="handlebars" @code={{"<p>Hello, {{@name}}!</p>"}} />`. A string literal
_inside_ a mustache is never processed for mustaches. (A quoted attribute value is:
`@code="Hello, {{@name}}"` would insert the value of `@name`.)

</details>

## Drawbacks

- Templates that use these forms have to change, even though they work today. The changes are
  mechanical, and Prettier makes most of them, but projects that do not use Prettier on templates
  have to edit them by hand.
- Addon templates that use these forms raise deprecations in apps that cannot fix them directly
  until the addon is updated.
- Prettier's output for `{{&foo}}` with whitespace control is wrong until Prettier is fixed, so
  "run Prettier" is not a complete fix for that one case.
- Build-time deprecations are easier to miss than runtime ones that report in the browser. This one is only visible at build time, unless we build a new mechanism for propagating syntax deprecation info to the browser. This RFC does not propose that, so it means the warnings are only in the terminal.

## Alternatives

- **Do nothing.** The forms keep working, and every tool that reads Ember templates keeps
  supporting syntax that Ember does not document, including the confusing inverse block and the
  half-raw raw block.
- **A lint rule only.** This steers new code away from the forms, but the parser and every other
  tool still have to support them.
- **Remove them without a deprecation.** They are rare, but removing working syntax without a
  deprecation cycle is against Ember's semver policy.
- **Build run-time reporting for purely syntactic deprecations**. Deprecations in the terminal are easier to miss and harder to manage than ones in the browser. These particular features are not visible to Ember at runtime, but we could build a new mechanism for propagating the warnings to the browser.

## Unresolved questions

- Whether other Handlebars-inherited quirks should be deprecated at the same time, for example
  that `{{else-thing}}` is an else section that invokes `-thing`, rather than a mustache named
  `else-thing`.
