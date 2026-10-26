---
stage: accepted
start-date: 2026-10-26T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - learning
prs:
  accepted:
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

# Deprecate the `mut` template helper

## Summary

Deprecate `{{mut}}` for removal in Ember 8. It is the write side of two-way bindings in templates, and its API docs already say "Don't use `mut`." This is the companion to [RFC #1248](https://github.com/emberjs/rfcs/pull/1248), which deprecates `readonly`.

## Motivation

`mut` turns a property path into a reference that can be written to. It has two uses:

```hbs
{{! A setter, through fn }}
<button {{on "click" (fn (mut this.isOpen) true)}}>Open</button>

{{! An explicit two-way binding into a classic component }}
{{my-input value=(mut this.name)}}
```

Neither use needs `mut`. A method on the backing class is a clearer setter. Arguments to classic components are already two-way bound when they are property paths, so `(mut this.name)` and `this.name` behave the same there.

`mut` also has problems of its own:

- `(mut @checked)` writes to the caller's property, which brings two-way bindings into Glimmer components. Their arguments are otherwise read-only.
- The same expression is a getter or a setter depending on where it appears. ember-template-lint's [`no-mut-helper`](https://github.com/ember-template-lint/ember-template-lint/blob/main/docs/rule/no-mut-helper.md) rule lists this as a reason to avoid it.
- Strict mode reserves it as a keyword (`STRICT_MODE_KEYWORDS` in `@ember/template-compiler`).

`mut` is public API, so it can only be removed at a major.

## Transition Path

### What is deprecated

Any use of the `mut` helper in a template, in both loose mode (`.hbs`) and strict mode (`<template>`). The deprecation fires at runtime when the helper is invoked, with:

```js
deprecate('The `mut` template helper is deprecated.', false, {
  id: 'deprecate-mut-helper',
  for: 'ember-source',
  since: { available: 'TBD', enabled: 'TBD' },
  until: '8.0.0',
  url: 'https://deprecations.emberjs.com/id/deprecate-mut-helper',
});
```

The [deprecation guide](#deprecation-guide) covers each use case.

### Ecosystem

- ember-template-lint already has [`no-mut-helper`](https://github.com/ember-template-lint/ember-template-lint/blob/main/docs/rule/no-mut-helper.md). It can move into the recommended config, and `no-extra-mut-helper-argument` can be retired after removal.
- A codemod can remove `mut` from classic component arguments. The setter case needs a backing class and `@tracked`, so it is not mechanical.
- After removal, `mut` leaves `STRICT_MODE_KEYWORDS`, so strict-mode templates can use the name for their own locals.
- Addons migrate the same way apps do.
- If Glint declares `mut` as a template global, that declaration goes away when the helper does.
- Blueprints (none generate `mut`), Ember Inspector, SSR / FastBoot, and Engines are not affected.

## How We Teach This

The guides do not teach `mut`. The work is:

- Open a PR to [ember-learn/deprecation-app](https://github.com/ember-learn/deprecation-app) with the guide below.
- Mark `mut` `@deprecated` in the API docs, with a link to the guide.

### Deprecation guide

The guide goes in `content/ember/v7/deprecate-mut-helper.md`. The folder follows the major version of `since`. The frontmatter is:

```yaml
---
title: The `mut` template helper
until: 8.0.0
since: TBD
rfc: https://rfcs.emberjs.com/id/0000-deprecate-mut-helper
---
```

The body is below, between the horizontal rules.

---

The `mut` template helper is deprecated. How you replace it depends on how you use it.

#### As a setter with `fn`

Replace `(fn (mut this.prop) value)` with a method on the backing class:

```gjs
// Before
export default class Dropdown extends Component {
  isOpen = false;

  <template>
    <button {{on "click" (fn (mut this.isOpen) true)}}>Open</button>
  </template>
}
```

```gjs
// After
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { on } from '@ember/modifier';

export default class Dropdown extends Component {
  @tracked isOpen = false;

  open = () => {
    this.isOpen = true;
  };

  <template>
    <button {{on "click" this.open}}>Open</button>
  </template>
}
```

Make sure the property is `@tracked`. `mut` writes through Ember's `set`, which updates the template even for untracked properties. A plain assignment does not.

#### On an argument, as in `(mut @value)`

This writes to the caller's property. Have the caller pass a callback instead:

```hbs
{{! Before: the child writes to the caller's property }}
<button {{on "click" (fn (mut @isOpen) false)}}>Close</button>

{{! After: the child asks the caller to change it }}
<button {{on "click" @onClose}}>Close</button>
```

```hbs
{{! The caller }}
<Panel @isOpen={{this.isOpen}} @onClose={{this.close}} />
```

#### On a classic component argument

Delete `mut`. Property paths passed to classic components are two-way bound without it.

```hbs
{{! Before }}
{{my-input value=(mut this.name)}}

{{! After }}
{{my-input value=this.name}}
```

#### As a getter

If the result of `(mut this.prop)` is only read, as in `{{#let (mut this.prop) as |prop|}}{{prop}}{{/let}}`, read `this.prop` directly.

---

## Drawbacks

- `(fn (mut this.prop))` was a common pattern in early Octane code. Each use needs a backing-class method, so the migration is larger than the `readonly` one.
- Template-only components that use `mut` need a backing class or a callback from the caller.

## Alternatives

- Keep `mut`. Its docs already discourage it, and it keeps two-way bindings available in Glimmer components.
- Ship a built-in setter helper, like [ember-set-helper](https://github.com/pzuraq/ember-set-helper), as the replacement. That keeps setters in templates but adds a new API to teach. It can be a separate RFC.
- Fold this into the classic component deprecation ([RFC #1216](https://github.com/emberjs/rfcs/pull/1216)). That does not fit, because `mut` also works in Glimmer components.

## Unresolved questions

n/a
