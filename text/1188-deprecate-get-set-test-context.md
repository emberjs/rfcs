---
stage: accepted
start-date: 2026-06-22T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - learning
  - typescript
prs:
  accepted: # update this to the PR that you propose your RFC in
project-link:
---

# Deprecate `this.get` and `this.set` on Test Contexts

## Summary

Deprecate the use of `this.get` and `this.set` on the test context object in rendering tests, as a logical conclusion of [RFC #785](https://github.com/emberjs/rfcs/blob/master/text/0785-remove-set-get-in-tests.md), which introduced `render(component)` and `rerender()` as the modern replacements.

## Motivation

RFC #785 introduced two new testing utilities — an updated `render` helper that accepts a component directly, and a new `rerender()` function — specifically to remove the need for `this.get` and `this.set` in rendering tests. Those APIs shipped in Ember v4.5.0 and are now the recommended approach for writing rendering tests.

The legacy pattern of setting values on `this` in a rendering test was problematic for several reasons laid out in RFC #785:

1. **Inconsistency with application code.** In post-Octane Ember, `get` and `set` are unnecessary; properties are tracked natively. Requiring them in tests is a confusing holdover.

2. **Incorrect rendering semantics.** `this.set` in tests is run-wrapped, causing a synchronous full DOM flush on every call. This does not reflect how Ember schedules DOM updates in production code, where changes to tracked state are coalesced.

3. **TypeScript friction.** Assigning arbitrary properties to `this` forces developers to redeclare the `TestContext` interface for every test module, causing leakage of property declarations across tests and defeating the purpose of static type checking.

Now that the modern replacements have been stable for multiple major versions, it is appropriate to deprecate the old approach and eventually remove it, completing the migration to a cleaner and more accurate testing model.

## Transition Path

### Before (deprecated)

```js
import { render } from '@ember/test-helpers';
import { hbs } from 'ember-cli-htmlbars';

test('it renders the name', async function (assert) {
  this.set('name', 'Zoey');

  await render(hbs`<MyComponent @name={{this.name}} />`);

  assert.dom('[data-test-name]').hasText(this.get('name'));

  this.set('name', 'Tomster');

  assert.dom('[data-test-name]').hasText(this.get('name'));
});
```

### After (recommended)

```js
import { render, rerender } from '@ember/test-helpers';
import { tracked } from '@glimmer/tracking';

test('it renders the name', async function (assert) {
  const state = new class {
    @tracked name = 'Zoey';
  };

  await render(<template><MyComponent @name={{state.name}} /></template>);

  assert.dom('[data-test-name]').hasText('Zoey');

  state.name = 'Tomster';
  await rerender();

  assert.dom('[data-test-name]').hasText('Tomster');
});
```

For projects that have not yet adopted `<template>` tag syntax, `precompileTemplate` from `@ember/template-compilation` can be used together with a scope hash instead:

```js
import { render, rerender } from '@ember/test-helpers';
import { precompileTemplate } from '@ember/template-compilation';
import { tracked } from '@glimmer/tracking';

test('it renders the name', async function (assert) {
  const state = new class {
    @tracked name = 'Zoey';
  };

  await render(precompileTemplate('<MyComponent @name={{state.name}} />', {
    scope: () => ({ state, MyComponent }),
  }));

  assert.dom('[data-test-name]').hasText('Zoey');

  state.name = 'Tomster';
  await rerender();

  assert.dom('[data-test-name]').hasText('Tomster');
});
```

### Deprecation message

When `this.set` or `this.get` is called on the test context, `@ember/test-helpers` should emit a deprecation warning of the form:

```
Using `this.set` / `this.get` on the test context is deprecated.
Please migrate to passing a component or template with a local tracked state object to `render()`,
and use `rerender()` to await DOM updates.
See https://deprecations.emberjs.com/id/test-context-get-set for details.
```

### Ecosystem considerations

- **`eslint-plugin-ember`** — A new lint rule (or an update to an existing rule) should be introduced to flag `this.set(…)` and `this.get(…)` calls inside `module(…)` / `test(…)` callbacks, guiding users toward the modern pattern.
- **Blueprints** — Any existing blueprints in `ember-source` or `ember-cli` that generate rendering-test boilerplate using `this.set`/`this.get` should be updated to use the component-based pattern.
- **Codemods** — A codemod (e.g. as part of `ember-codemods` or a standalone package) should be provided to automate the majority of the migration. Cases where the test drives complex state that touches multiple `this.set` calls will require manual attention, but simple single-value cases should be fully automatable.

## How We Teach This

The [Testing Components](https://guides.emberjs.com/release/testing/testing-components/) section of the official guides should be updated to:

1. Remove all examples that use `this.set` / `this.get`.
2. Present the `render(component)` + `rerender()` pattern as the canonical approach.
3. Include a brief migration note linking to the deprecation guide on `deprecations.emberjs.com`.

The API docs for `@ember/test-helpers` should likewise be updated to mark `TestContext#set` and `TestContext#get` as deprecated and point to the alternatives.

A deprecation guide entry should be added to `deprecations.emberjs.com` covering:

- What was deprecated and why.
- The before/after code examples above.
- Guidance for the less common edge cases (e.g. deeply nested `this.set` calls, helper-heavy templates).

## Drawbacks

- **Migration cost.** `this.set` and `this.get` have been in use since the beginning of Ember's testing story. Large codebases may have hundreds or thousands of test files that need updating. A well-maintained codemod should substantially reduce the burden.
- **Loss of a simple entry point.** For simple tests, `this.set('value', x)` is arguably terser than introducing a tracked class. This cost is outweighed by the correctness and TypeScript benefits, but should be acknowledged in migration documentation.

## Alternatives

- **Do nothing.** Leave `this.set` / `this.get` available indefinitely. This leaves the inconsistency and TypeScript friction in place and means RFC #785's goal of a cleaner test model is never fully realized.
- **Hard removal without a deprecation period.** Would be a breaking change and is not appropriate given how widespread the pattern is.

## Unresolved questions

- What is the appropriate Ember version for the deprecation to land in, and what is the target removal version (i.e. the next major)?
- Should the deprecation be opt-in (gated by a flag in `@ember/test-helpers` config) initially, to allow teams to migrate on their own schedule, before becoming unconditional?
- Should `this.owner` and other non-`get`/`set` test-context properties remain unaffected? (Yes — this RFC is scoped only to `get` and `set`.)
