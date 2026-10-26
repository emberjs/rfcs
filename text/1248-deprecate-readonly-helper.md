---
stage: accepted
start-date: 2026-10-02T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1248
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

# Deprecate the `readonly` template helper

## Summary

`{{readonly}}` affects only classic components, which [RFC #1216](https://github.com/emberjs/rfcs/pull/1216) deprecates. The API docs have marked it `@private` since 2016, but it was never deprecated and it still works. That makes it intimate API. This RFC deprecates the helper so that Ember can remove it after the next LTS.

## Motivation

`readonly` lets the caller opt out of two-way bindings when it passes a value into a classic component (`import Component from '@ember/component'`):

```hbs
{{my-child count=(readonly this.total)}}
<MyChild @count={{readonly this.total}} />
```

Without `readonly`, a classic component that calls `this.set('count', ...)` writes the new value back to the caller. With `readonly`, the write stays local to the child.

That is all `readonly` does. Glimmer components (`@glimmer/component`) and template-only components cannot set their arguments, so `readonly` does nothing for them. It matters only for classic components, and [RFC #1216](https://github.com/emberjs/rfcs/pull/1216) deprecates them.

`readonly` still has a cost:

- It is one of three public names that strict mode still treats as a keyword (`STRICT_MODE_KEYWORDS` in `@ember/template-compiler`), alongside `mut` and `unbound`.
- Its status is unclear. The API docs have hidden it since 2016 (refer to [emberjs/ember.js#21549](https://github.com/emberjs/ember.js/issues/21549)), but it still works and still appears in older apps and addons. Developers who find it in a codebase have no documentation and no guidance on whether to keep it.

## Transition Path

### What is deprecated

Any use of the `readonly` helper in a template, in both loose mode (`.hbs`) and strict mode (`<template>`). The deprecation fires at runtime the first time the helper is invoked, with:

```js
deprecate('The `readonly` template helper is deprecated.', false, {
  id: 'deprecate-readonly-helper',
  for: 'ember-source',
  since: { available: '<the release that ships it>', enabled: '<the same release>' },
  until: '<the first LTS after it is enabled>',
  url: 'https://deprecations.emberjs.com/id/deprecate-readonly-helper',
});
```

`available` and `enabled` are the same version because the API is intimate. Ember's intimate API policy allows removal after one LTS cycle, so this deprecation does not need to wait for the next major.

### Use cases

Each use of `readonly` falls into one of three cases, based on the component that receives the value. The [deprecation guide](#deprecation-guide) below gives the steps for each.

1. **Glimmer or template-only component.** `readonly` does nothing. Delete it.
2. **Classic component that never sets the property.** `readonly` does nothing in practice. Delete it.
3. **Classic component that sets the property.** Give the child its own state, or pass a callback, then delete `readonly`. Until then, an identity helper keeps the current behavior.

### Ecosystem

- **ember-template-lint**: add a `no-readonly` rule (or extend an existing deprecated-helper rule) that flags the helper, with an autofix that removes it when the invoked component is resolvable and is not a classic component. A codemod can apply the same fix across a codebase.
- **Strict mode**: once removed, `readonly` is dropped from `STRICT_MODE_KEYWORDS`, freeing the name for user locals.
- **Addons**: addons that still ship classic components and pass `readonly` values will see the deprecation in their consumers' apps.
- **Glint**: if Glint declares `readonly` as a template global, remove that declaration when Ember removes the helper.
- **No impact**: blueprints (none generate `readonly`), Ember Inspector, SSR / FastBoot, Engines.

## How We Teach This

`readonly` is not in the guides, so there is nothing to remove from the teaching material. The work is:

- Open a PR to [ember-learn/deprecation-app](https://github.com/ember-learn/deprecation-app) that adds the deprecation guide below, so it appears at [deprecations.emberjs.com](https://deprecations.emberjs.com/id/deprecate-readonly-helper).
- Keep the existing API doc entry `@private`, and add a `@deprecated` note that points to the guide.

### Deprecation Guide

The guide goes in `content/ember/v7/deprecate-readonly-helper.md`. The folder follows the major version of `until`. The frontmatter is:

```yaml
---
title: The `readonly` template helper
until: <the first LTS after it is enabled>
since: <the release that ships it>
rfc: https://rfcs.emberjs.com/id/1248-deprecate-readonly-helper
---
```

The body is below, between the horizontal rules.

---

The `readonly` template helper is deprecated. It has an effect only when you pass a value to a classic component (`@ember/component`). There, it stops the child from writing changes back to the caller through a two-way binding.

#### Glimmer components and template-only components

Delete `readonly`. Arguments to these components are already read-only.

```hbs
{{! Before }}
<Counter @count={{readonly this.total}} />

{{! After }}
<Counter @count={{this.total}} />
```

#### Classic components

Check whether the component ever sets the property it receives. Look in its class (`this.set('count', ...)`, `this.incrementProperty('count')`) and in its template (`{{mut this.count}}`, `<Input @value={{this.count}} />`).

- If it never does, delete `readonly`.
- If it does, give the child its own local state instead of writing to the passed-in property, then delete `readonly`:

```js
// Before: relies on the caller passing (readonly this.total)
export default class Counter extends Component {
  click() {
    this.incrementProperty('count');
  }
}

// After: the child keeps its own copy
export default class Counter extends Component {
  localCount = this.count;

  click() {
    this.incrementProperty('localCount');
  }
}
```

Then change the child's template to read `this.localCount` instead of `this.count`. `localCount` is a copy taken when the child is created. With `readonly`, later changes to `this.total` in the caller still reached the child. If the child must follow those changes, pass it a callback that updates the caller's state instead. The [Octane vs Classic cheat sheet](https://ember-learn.github.io/ember-octane-vs-classic-cheat-sheet/) shows this pattern (Data Down, Actions Up).

Converting the child to a Glimmer component, the long-term path under the [classic component deprecation](https://deprecations.emberjs.com/id/ember-component), needs the same change.

#### If you cannot change the child yet

Wrap the value in an identity helper. Helper results are read-only references, so this keeps the current behavior:

```js
// app/helpers/one-way.js
export default function oneWay(value) {
  return value;
}
```

```hbs
{{my-child count=(one-way this.total)}}
```

---

## Drawbacks

- Apps with many classic components will see more deprecations, in addition to the deprecations for classic components themselves.
- Deleting `readonly` from a classic component invocation without checking the child can silently turn a one-way binding back into a two-way one. The guide's classic component section and the lint autofix's classic exclusion handle this.

## Alternatives

- **Restore the API docs to public** and keep the helper. This documents a feature that only serves classic components, which are themselves being deprecated, and keeps a strict-mode keyword reserved for it indefinitely.
- **Fold it into the classic component deprecation** ([RFC #1216](https://github.com/emberjs/rfcs/pull/1216)) and remove it at the next major. This waits longer than the intimate API policy requires. A separate deprecation also gives it its own id and guide, instead of one line in a much larger migration.
- **Do nothing.** The helper stays undocumented and reserved as a keyword. RFC #1216 removes classic components, but nothing removes `readonly`, so it outlives the only components it affects.

## Unresolved questions

- Which version number goes in `until`? This RFC proposes the first LTS after the deprecation is enabled.
- Should `mut` get the same treatment in a companion RFC? `mut` is the write side of the same two-way binding system. It also works as a setter with `fn` (`{{fn (mut this.value)}}`) in any component type, so a companion RFC needs a migration path for that use too.
