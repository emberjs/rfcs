---
stage: accepted
start-date: 2026-10-02T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
prs:
  accepted: # update this to the PR that you propose your RFC in
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

Deprecate the `{{readonly}}` template keyword. It has been marked `@private` in the API docs since 2016, but it was never deprecated and still works, which makes it intimate API. This RFC deprecates it so it can be removed after the next LTS.

## Motivation

`readonly` exists to opt out of two-way bindings when passing a value into a classic component (`import Component from '@ember/component'`):

```hbs
{{my-child count=(readonly this.total)}}
<MyChild @count={{readonly this.total}} />
```

Without `readonly`, a classic component that calls `this.set('count', ...)` writes the new value back up to the caller. With it, the write stays local to the child.

That is the only thing `readonly` does, and it only matters for classic components:

- Glimmer components (`@glimmer/component`) and template-only components cannot set their arguments, so `readonly` is a no-op for them.
- Plain functions used as helpers, and any other helper result, are already read-only references.
- [RFC #1216](https://github.com/emberjs/rfcs/pull/1216) deprecates the classic component class, which is the last thing in the framework that supports two-way bindings.

Meanwhile `readonly` still costs something:

- It is one of the few built-in keywords still recognized in strict mode (`STRICT_MODE_KEYWORDS` in `ember-template-compiler`), alongside `mut` and `unbound`.
- It sits in an odd state: hidden from the API docs since 2016 ([emberjs/ember.js#21549](https://github.com/emberjs/ember.js/issues/21549)), yet working, untaught, and still showing up in older apps and addons. Developers who find it in a codebase have no documentation and no guidance on whether to keep it.

Deprecating it resolves that state and lets the keyword be removed along with the rest of the two-way binding machinery.

## Transition Path

### What is deprecated

Any use of the `readonly` keyword in a template, in both loose mode (`.hbs`) and strict mode (`<template>`). The deprecation fires at runtime the first time the helper is invoked, with:

- id: `deprecate-readonly-helper`
- for: `ember-source`
- since: the release that ships it (`available` and `enabled` together, since the API is intimate)
- until: the next LTS release after it is enabled (see Unresolved questions)
- url: `https://deprecations.emberjs.com/id/deprecate-readonly-helper`

Ember's intimate API policy allows removal after one LTS cycle, so this deprecation does not need to wait for the next major.

### Deprecation guide

> The `readonly` template helper is deprecated. It only had an effect when passing a value to a classic component (`@ember/component`), where it stopped the child from writing changes back to the caller through a two-way binding.
>
> **If the receiving component is a Glimmer component or template-only component**, delete `readonly`. Arguments to those components are already read-only.
>
> ```hbs
> {{! Before }}
> <Counter @count={{readonly this.total}} />
>
> {{! After }}
> <Counter @count={{this.total}} />
> ```
>
> **If the receiving component is a classic component**, check whether it ever sets the property it receives (`this.set('count', ...)`, `this.incrementProperty('count')`, `{{mut count}}`, `<Input @value={{this.count}} />`, and so on).
>
> - If it never does, delete `readonly`.
> - If it does, give the child its own local state instead of writing to the passed-in property, and delete `readonly`:
>
> ```js
> // Before: relies on the caller passing (readonly total)
> export default class Counter extends Component {
>   click() {
>     this.incrementProperty('count');
>   }
> }
>
> // After: the child keeps its own copy
> export default class Counter extends Component {
>   localCount = this.count;
>
>   click() {
>     this.incrementProperty('localCount');
>   }
> }
> ```
>
> Converting the child to a Glimmer component, which is the long-term path under the classic component deprecation, has the same effect.
>
> If neither change is possible right away, any helper result is a read-only reference, so an identity function keeps the current behavior:
>
> ```js
> // app/helpers/read-only.js
> export default function readOnly(value) {
>   return value;
> }
> ```
>
> ```hbs
> {{my-child count=(read-only this.total)}}
> ```

### Ecosystem

- **ember-template-lint**: add a `no-readonly` rule (or extend an existing deprecated-helper rule) that flags the keyword, with an autofix that removes it when the invoked component is resolvable and is not a classic component.
- **Codemod**: removing `(readonly x)` → `x` is mechanical. The only case needing review is a classic component that writes to the property, which a codemod can flag but should not change.
- **Strict mode**: once removed, `readonly` is dropped from `STRICT_MODE_KEYWORDS`, freeing the name for user locals.
- **Addons**: addons that still ship classic components and pass `readonly` values will see the deprecation in their consumers' apps. The transition above applies to them unchanged.

## How We Teach This

`readonly` is not in the guides and has been hidden from the API docs since 2016, so there is nothing to remove from the teaching material. The work is:

- publish the deprecation guide above at deprecations.emberjs.com
- keep the existing API doc entry `@private`, and add a `@deprecated` note pointing at the guide, so people who find it in a codebase can learn what to do

Two-way bindings are already taught as a classic-component-only concept, and the guides already teach passing callbacks instead of mutating arguments.

## Drawbacks

- Apps with many classic components will see more deprecation noise, on top of the deprecations for classic components themselves.
- Deleting `readonly` from a classic component invocation without checking the child can silently turn a one-way binding back into a two-way one. The deprecation guide and lint rule need to call this out clearly.

## Alternatives

- **Restore the API docs to public** and keep the helper. This documents a feature that only serves classic components, which are themselves being deprecated, and keeps a strict-mode keyword reserved for it indefinitely.
- **Fold it into the classic component deprecation** ([RFC #1216](https://github.com/emberjs/rfcs/pull/1216)) and remove it at the next major. This works, but `readonly` is intimate API and can be removed sooner, and a separate deprecation gives it its own id and guide instead of being one line in a much larger migration.
- **Do nothing.** The helper stays undocumented, working, and reserved as a keyword.

## Unresolved questions

- Which LTS should `until` name? The proposal is the first LTS after the deprecation is enabled.
- Should `mut` get the same treatment in a companion RFC? It is the write side of the same two-way binding system and is also only meaningful with classic components.
