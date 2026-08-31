---
stage: accepted
start-date: 2026-06-22T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - learning
  - cli
  - typescript
prs:
  accepted: # update this to the PR that you propose your RFC in
project-link:
---

# Deprecate Loose Mode and HBS File Authoring

⚠️  Placeholder RFC generated with copilot, probably don't bother reading yet. 

## Summary

Deprecate loose mode Handlebars templates and authoring components and route
templates in `.hbs` files, in favor of the strict mode authoring experience
provided by `.gjs` and `.gts` template tag files. This is the natural
conclusion of [RFC #0496 (Handlebars Strict Mode)][rfc-0496],
[RFC #0779 (First-Class Component Templates)][rfc-0779], and
[RFC #1046 (Template Tag in Routes)][rfc-1046] all reaching the
[recommended][stages] stage.

[rfc-0496]: https://github.com/emberjs/rfcs/blob/master/text/0496-handlebars-strict-mode.md
[rfc-0779]: https://github.com/emberjs/rfcs/blob/master/text/0779-first-class-component-templates.md
[rfc-1046]: https://github.com/emberjs/rfcs/blob/master/text/1046-template-tag-in-routes.md
[stages]: https://github.com/emberjs/rfcs/blob/master/README.md#stages

## Motivation

Ember's template language has two modes:

- **Loose mode** (also called non-strict or classic mode): the default behavior
  used by standalone `.hbs` files. It resolves components, helpers, and
  modifiers dynamically by name at runtime using the resolver. It supports
  implicit `this` lookup, implicit globals, and dynamic resolution.

- **Strict mode**: the semantics defined in [RFC #0496][rfc-0496], where every
  value in a template must be explicitly in scope. There are no implicit globals,
  no implicit `this` fallback, no dynamic resolution, and no `eval`-like
  features (e.g., partials, which were deprecated separately). Strict mode is
  the only mode available in `.gjs` and `.gts` template tag files.

### The Recommended Path Is Already Clear

All three foundation RFCs have progressed to recommended:

- [RFC #0496][rfc-0496] made strict mode semantics the recommended way to write
  templates.
- [RFC #0779][rfc-0779] made `.gjs`/`.gts` template tag files—which are always
  strict mode—the recommended authoring format for components.
- [RFC #1046][rfc-1046] extended template tag authoring to route templates,
  completing coverage of every template in an Ember application.

Additionally, [RFC #0995 (Deprecate Non-Co-located Components)][rfc-0995]
already deprecated the classic separate-directory and pods component layouts,
leaving only co-located `.hbs`+`.js`/`.ts` pairs and `.gjs`/`.gts` files as
supported component authoring formats.

[rfc-0995]: https://github.com/emberjs/rfcs/blob/master/text/0995-deprecate-non-colocated-components.md

The ecosystem has converged. New Ember apps generated with the current blueprint
default to Embroider+Vite, which has first-class `.gjs`/`.gts` support. The
community is actively producing codemods, lint rules, and IDE tooling for
`.gjs`/`.gts`. Maintaining loose mode indefinitely imposes ongoing costs:

- **Implementation cost**: the resolver logic, implicit lookup behavior, and
  loose mode compilation paths must be maintained alongside the stricter,
  simpler strict mode path.

- **Teaching cost**: new Ember users encounter two incompatible template
  authoring worlds—global-resolution `.hbs` files and import-based `.gjs`
  files—which creates confusion about which rules apply where.

- **Tooling cost**: language servers, linters, and formatters must handle two
  distinct template semantics. Strict mode has richer static analysis
  possibilities (e.g., "go to definition" for components) that cannot be
  offered in loose mode.

- **Performance cost**: loose mode requires a live resolver at runtime to
  dynamically look up components, helpers, and modifiers by name. Strict mode
  can be fully resolved at build time.

### What Is Being Deprecated

This RFC deprecates:

1. **Loose mode compilation and runtime resolution** — the mechanism by which
   `.hbs` templates resolve component, helper, and modifier names through
   implicit global lookup and `this` fallback.

2. **`.hbs` files for component templates** — co-located `.hbs` files paired
   with a `.js` or `.ts` backing class (e.g., `my-component.hbs` alongside
   `my-component.js`), as well as template-only `.hbs` files.

3. **`.hbs` files for route templates** — files under `app/templates/` that are
   used as route-level templates (e.g., `app/templates/application.hbs`).

This RFC does **not** deprecate:

- The `.hbs` file extension itself in other contexts (e.g., inline test
  templates using `hbs` tagged template literals in test files remain supported
  and are a separate concern).
- The underlying Glimmer VM; only the loose mode compilation path on top of it
  is deprecated.
- Any `.gjs`/`.gts` behavior; those files are the recommended replacement.

## Transition Path

### Components

**Before (co-located hbs + js):**
```js
// app/components/my-greeting.js
import Component from '@glimmer/component';

export default class MyGreeting extends Component {}
```

```hbs
{{! app/components/my-greeting.hbs }}
<p>Hello, {{@name}}!</p>
```

**After (template tag gjs):**
```gjs
// app/components/my-greeting.gjs
import Component from '@glimmer/component';

export default class MyGreeting extends Component {
  <template>
    <p>Hello, {{@name}}!</p>
  </template>
}
```

**Before (template-only hbs component):**
```hbs
{{! app/components/my-greeting.hbs }}
<p>Hello, {{@name}}!</p>
```

**After (template-only gjs component):**
```gjs
{{! app/components/my-greeting.gjs }}
<template>
  <p>Hello, {{@name}}!</p>
</template>
```

### Route Templates

**Before (route hbs template):**
```hbs
{{! app/templates/posts.hbs }}
<h1>Posts</h1>
{{#each @model as |post|}}
  <Post @post={{post}} />
{{/each}}
```

**After (route gjs template):**
```gjs
// app/templates/posts.gjs
import Post from 'my-app/components/post';

<template>
  <h1>Posts</h1>
  {{#each @model as |post|}}
    <Post @post={{post}} />
  {{/each}}
</template>
```

Note that in the `.gjs` route template, components must be explicitly imported.
Embroider provides a codemod that can automatically add the necessary imports
when migrating from loose mode templates.

### Deprecation Warnings

A runtime deprecation warning should be emitted when a loose mode template is
compiled and invoked. The deprecation message should link to this RFC and to the
[deprecation guide][deprecation-guide].

[deprecation-guide]: https://deprecations.emberjs.com/

The deprecation ID will be `ember.loose-mode-templates`.

For co-located `.hbs` component templates, the warning should identify the
specific component file that triggered it (e.g.,
`app/components/my-greeting.hbs`).

For route-level `.hbs` templates, the warning should identify the route and
template file.

### Codemods

The [ember-codemod-template-tag][codemod] package can automatically convert
most `.hbs` component and route template files to `.gjs`/`.gts`. It handles:

- Merging a co-located `.hbs` + `.js`/`.ts` pair into a single `.gjs`/`.gts` file.
- Converting template-only `.hbs` components to template-only `.gjs` files.
- Converting route templates from `.hbs` to `.gjs`.
- Adding explicit imports for components, helpers, and modifiers that were
  previously resolved by name.

[codemod]: https://github.com/IgnaceMaes/ember-codemod-template-tag

Embroider also provides infrastructure for generating the necessary imports
during migration as part of its compatibility layer.

### Lint Rules

#### ember-template-lint

A new `no-loose-mode-templates` rule (or equivalent) should be added to
[ember-template-lint][etl] that reports any `.hbs` file used as a component or
route template as a warning (later, an error). This gives projects a lint-time
signal before the deprecation becomes a runtime error in a future major release.

[etl]: https://github.com/ember-template-lint/ember-template-lint

#### eslint-plugin-ember

The [eslint-plugin-ember][epe] `no-classic-components` rule (or an analogous
rule) should be updated or supplemented to flag co-located `.hbs` files
alongside their `.js`/`.ts` backing classes, encouraging migration to
`.gjs`/`.gts`.

[epe]: https://github.com/ember-cli/eslint-plugin-ember

### Blueprints

All relevant `ember-cli` blueprints should be updated to generate `.gjs` or
`.gts` files by default:

- `ember generate component` — generate `component.gjs`/`component.gts` instead
  of separate `component.hbs` + `component.js`/`component.ts` files.
- `ember generate route` — generate `route.gjs`/`route.gts` for the route
  template instead of a standalone `.hbs` file.

These changes follow the blueprints update already made in
[RFC #1055 (Vanilla Prettier Setup in Blueprints)][rfc-1055] and related
blueprint work.

[rfc-1055]: https://github.com/emberjs/rfcs/blob/master/text/1055-vanilla-prettier-setup-in-blueprints.md

### Addon Ecosystem

Addons that ship `.hbs` component templates should migrate to one of:

- **V2 addon format** with `.gjs`/`.gts` files (preferred, per
  [RFC #0507][rfc-0507] and [RFC #0985][rfc-0985]).
- Co-located `.hbs` files will continue to work through the deprecation
  period (one full major version cycle) and will emit a deprecation warning
  so addon authors are aware.

[rfc-0507]: https://github.com/emberjs/rfcs/blob/master/text/0507-embroider-v2-package-format.md
[rfc-0985]: https://github.com/emberjs/rfcs/blob/master/text/0985-v2-addon-by-default.md

Addon authors should update their published packages before the loose mode
removal in the next Ember major version.

### Timeline

- **Deprecation period**: loose mode templates emit a deprecation warning
  starting in the Ember minor release where this RFC is implemented.
- **Removal**: loose mode template support will be removed in the next Ember
  major version following the deprecation release, consistent with Ember's
  standard deprecation policy.

## How We Teach This

### Guides

The Ember Guides should be updated to exclusively use `.gjs`/`.gts` examples
in all component and route template sections. References to `.hbs` file
authoring should be moved to a "migrating from classic templates" section, not
featured in the primary learning path.

The tutorial should already be using template tag syntax; if it isn't, this
deprecation is a forcing function to do so.

### Deprecation Guide

A deprecation guide should be published at
[deprecations.emberjs.com](https://deprecations.emberjs.com/) with the
deprecation ID `ember.loose-mode-templates`. It should cover:

- An explanation of loose mode and why it is being deprecated.
- Step-by-step migration instructions for:
  - Co-located hbs + js/ts component pairs.
  - Template-only hbs components.
  - Route-level hbs templates.
- How to run the codemod.
- How to silence the deprecation while migrating (using
  `@ember/optional-features` or a similar mechanism if one is provided).

### API Docs

API documentation that references loose mode resolution, the classic resolver,
or `.hbs` template authoring should be updated to link to the deprecation guide
and the template tag authoring docs.

### Messaging to Existing Users

When the deprecation is introduced:

- A blog post should announce the deprecation, explain the rationale, link to
  the deprecation guide, and highlight codemod tooling.
- The Ember CLI `ember generate` command should emit a one-time notice directing
  users to migrate existing `.hbs` files when it detects them in a project.

## Drawbacks

- **Ecosystem churn**: many addons and apps still use `.hbs` files. The migration
  requires running a codemod or manually converting each file. For large
  applications with hundreds of templates, this can be a significant one-time
  investment.

- **Addon consumers**: end users of addons that have not migrated will see
  deprecation warnings that they cannot suppress without forking the addon or
  waiting for the addon author to publish an update.

- **`.hbs`-only workflows**: some teams use external templating tools or
  workflows that specifically target `.hbs` files. Those workflows will need to
  adapt.

- **Incremental adoption**: projects that cannot immediately migrate all files
  will need to live with deprecation warnings for a potentially extended period.

## Alternatives

### Keep Loose Mode Indefinitely

We could accept that loose mode is a permanent part of Ember and never deprecate
it. This avoids migration burden but accepts all the ongoing costs described in
the Motivation section: the implementation, teaching, tooling, and performance
costs of maintaining two template systems in perpetuity.

### Deprecate Loose Mode Semantics Without Deprecating `.hbs` Files

We could separately deprecate the loose mode *behaviors* (implicit globals,
implicit `this` fallback) while still allowing `.hbs` files that have been
compiled with explicit strict mode flags. In practice, all `.hbs` files in Ember
apps currently use loose mode, and the ergonomics of opting into strict mode in
a standalone `.hbs` file are poor. This alternative would add complexity without
a clear benefit over moving entirely to `.gjs`/`.gts`, which provide strict mode
by design.

### Require Strict Mode in `.hbs` Files

We could require `.hbs` files to opt into strict mode rather than deprecating
them entirely. This is a less-breaking path but perpetuates the existence of a
second template format alongside `.gjs`/`.gts`, splitting the ecosystem without
delivering the simplification we are aiming for.

## Unresolved questions

- Should there be an opt-out mechanism (e.g., an optional feature flag) to
  silence the deprecation warning globally for projects that need more time to
  migrate? Or should the deprecation always emit to ensure visibility?

- What is the minimum Ember version at which this deprecation can land, given
  that RFC #1046 (template tag in routes) landed in Ember 6.3? Should this
  deprecation target Ember 6.x or wait for 7.x?

- Should `.hbs` files inside `tests/` (e.g., integration test helper templates
  authored with the `hbs` tagged template literal) be in scope for this
  deprecation, or handled in a separate RFC?

- How should Ember Engines handle this transition? Engines may have their own
  template namespaces and resolver configurations; the deprecation messaging
  should address this.
