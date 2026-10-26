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

`{{readonly}}` affects only classic components, which [RFC #1216](https://github.com/emberjs/rfcs/pull/1216) deprecates. It has been `@private` since 2016 but was never deprecated, which makes it intimate API. This RFC deprecates it for removal after the next LTS.

## Motivation

`readonly` lets the caller opt out of two-way bindings when it passes a value into a classic component (`import Component from '@ember/component'`):

```hbs
{{my-child count=(readonly this.total)}}
<MyChild @count={{readonly this.total}} />
```

Without `readonly`, a classic component that calls `this.set('count', ...)` writes the new value back to the caller. With `readonly`, the write stays local to the child.

Glimmer components (`@glimmer/component`) and template-only components cannot set their arguments, so `readonly` does nothing for them.

`readonly` still has a cost:

- It is one of three public names that strict mode still treats as a keyword (`STRICT_MODE_KEYWORDS` in `@ember/template-compiler`), alongside `mut` and `unbound`.
- It is undocumented but still appears in older apps and addons, so developers who find it have no guidance ([emberjs/ember.js#21549](https://github.com/emberjs/ember.js/issues/21549)).

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

Because the API is intimate, `available` and `enabled` match, and `until` is the next LTS instead of the next major.

The [deprecation guide](#deprecation-guide) covers each use case.

### Ecosystem

- **ember-template-lint**: add a `no-readonly` rule with an autofix that removes it when the invoked component is resolvable and is not a classic component. A codemod can apply the same fix across a codebase.
- **Strict mode**: once removed, `readonly` is dropped from `STRICT_MODE_KEYWORDS`, freeing the name for user locals.
- **Addons**: same migration as apps.
- **Glint**: if Glint declares `readonly` as a template global, remove that declaration when Ember removes the helper.
- **No impact**: blueprints (none generate `readonly`), Ember Inspector, SSR / FastBoot, Engines.

## How We Teach This

`readonly` is not in the guides. The work is:

- Open a PR to [ember-learn/deprecation-app](https://github.com/ember-learn/deprecation-app) with the guide below.
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
- If it does, give the child its own local state, then delete `readonly`:

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

Then change the child's template to read `this.localCount` instead of `this.count`. `localCount` is a copy taken at creation, so later changes to `this.total` no longer reach the child. If the child must follow them, pass it a callback that updates the caller's state ([Data Down, Actions Up](https://ember-learn.github.io/ember-octane-vs-classic-cheat-sheet/)).

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

- Apps with many classic components get more deprecations on top of RFC #1216's.
- Deleting `readonly` from a classic component invocation without checking the child can silently turn a one-way binding back into a two-way one. The guide and the lint autofix both account for this.

## Alternatives

- **Restore the API docs to public** and keep the helper. This documents a feature whose only use RFC #1216 removes.
- **Fold it into the classic component deprecation** ([RFC #1216](https://github.com/emberjs/rfcs/pull/1216)) and remove it at the next major. This waits longer than needed and buries the migration in a much larger guide.
- **Do nothing.** `readonly` outlives the classic components it serves, as an undocumented reserved keyword.

## Unresolved questions

- Which version number goes in `until`? This RFC proposes the first LTS after the deprecation is enabled.
- Should `mut` get the same treatment in a companion RFC? `mut` is the write side of the same system, but it also works as a setter with `fn` (`{{fn (mut this.value)}}`) in any component, so it needs its own migration path.
