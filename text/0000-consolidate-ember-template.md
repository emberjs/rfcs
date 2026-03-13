---
stage: accepted
start-date: 2026-02-28
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1170 
project-link:
suite: 
---

# Consolidate Template Concerns into `@ember/template`

## Summary

Provide everything for working on templates in Ember from _one_ place: `@ember/template`

## Motivation

Before gts/gjs files (polaris), the ember resolver strictly forced components to come from
the `components/` directory. That also constrained developers to limit their
thinking to the technical aspect of a "component" as a citizen/building block of
Ember.

With the era of gts/gjs (polaris) components can live freely in the codebase
within the boundary of a domain concern. gts/gjs files are also used
for routes, shifting the thinking from components as a technical aspect to
templates as the technical aspect; components can now be defined within a route
file.

Ember's code around templates is very much fragmented, due to experimentation
and excourses to glimmer and glint. If you are working on template these days,
you import from these places:

- `import Component from '@glimmer/component'`
- `import type { TOC } from '@ember/component/template-only'`
- `import Modifier, { modifier } from 'ember-modifier'`
- `import Helper, { helper } from '@ember/component/helper'`
- `import { on } from '@ember/modifier'`
- `import { fn, hash, array, concat, get, uniqueId } from '@ember/helper'`
- `import Component from '@ember/component'` (classic)
- `import * from '@glint/template'`
- `import { htmlSafe, isHTMLSafe, trustHTML, isTrustedHTML, type SafeString, type TrustedHTML } from '@ember/template'`

Plus there are "template building block" exports in the respective "family
module" (for completeness on the API):

- `import { capabilities, setHelperManager, invokeHelper } from '@ember/helper'`
- `import { capabilities, setComponentManager, setComponentTemplate, getComponentTemplate } from '@ember/component'`

Many of them is needed on a daily basis for working with Ember. This is a lot to
remember, import statements in the beginnig of files grow and due to its
fragmentation it's hard to understand which of these imports have a coherent "belonging".

## Detailed design

The idea is to bring everything concerning template citizens/building blocks
together and make it available from _one_ location: `@ember/template`. That will
also be the place for developing previously RFC'd built-in helpers/modifiers: [`(element)`](https://rfcs.emberjs.com/id/0389-dynamic-tag-names/),
[`{{on}}`](https://rfcs.emberjs.com/id/0997-make-on-built-in/),
[`(fn)`](https://rfcs.emberjs.com/id/0998-make-fn-built-in/),
[`(hash)`](https://rfcs.emberjs.com/id/0999-make-hash-built-in/),
[`(array)`](https://rfcs.emberjs.com/id/1000-make-array-built-in/), [logical
operators](https://rfcs.emberjs.com/id/0562-add-logical-operators/), [comparison
operators](https://rfcs.emberjs.com/id/0561-add-numeric-comparison-operators/),
[equality operators](https://rfcs.emberjs.com/id/0560-add-equality-operators/).

### New Imports / Public API

The new public API for template concerns as per this RFC:

| Old Import | New Import (`... from '@ember/template'`) | Notes |
| ---------- | ----------------------------------------- | ----- |
| `import Component from '@glimmer/component'` | `import { Component }` | |
| `import type { TOC } from '@ember/component/template-only'` | `import type { Component }` | |
| `import Modifier, { modifier } from 'ember-modifier'` | `import { modifier, Modifier }` | |
| `import Helper, { helper } from '@ember/component/helper'` | `import { helper, Helper }` | |
| `import { on } from '@ember/modifier'` | `import { on }` | [RFC 997](https://rfcs.emberjs.com/id/0997-make-on-built-in/) |
| `import { concat, get, uniqueId } from '@ember/helper'` | `import { concat, get, uniqueId }` | |
| `import { fn, hash, array } from '@ember/helper'` | `import { fn, hash, array }` | [RFC 998](https://rfcs.emberjs.com/id/0998-make-fn-built-in/), [RFC 999](https://rfcs.emberjs.com/id/0999-make-hash-built-in/), [RFC 1000](https://rfcs.emberjs.com/id/1000-make-array-built-in/) |
| `import * from '@glint/template'` | `import *` | All template types move |

The old imports will be marked as deprecated and dropped at the next major release.

## How we teach this

Put deprecation comments on outdated imports with instructions what to do. This
is the best way to reach the fingertips of developers.

Mention in the release post, when the new imports are available.

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Ember,
on the integration of this feature with other existing and planned features,
on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

Instead of merging everything into `@ember/templates`, all the imports can go
into their respective families: `@ember/component`, `@ember/modifier` and
`@ember/helper`. Our API surface is already too big, this would keep it that
way, it also comes with conflicts: where/what to do with `Component`?

## Unresolved questions

### Type Sanitation ?

Over the past years, we've become more and more comfortable with types around
glimmer components and also use them for linting. They affect the signatures of
helpers, modifiers and components and the are powered by the types that nowadays
reside in `@glint/template`. They are designed to be private but extensible to
some extend (to patch uncovered cases).

While the situation improved over time, there are still some gotchas that can be
patched with that move. Shall that happen as part of this RFC?
