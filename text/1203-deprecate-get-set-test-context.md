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
  accepted: https://github.com/emberjs/rfcs/pull/1203
project-link:
---

# Deprecate `get`, `set`, `getProperties`, and `setProperties` on test contexts

## Summary

Deprecate the four data-manipulation methods that `@ember/test-helpers` installs on the test context: `this.get`, `this.set`, `this.getProperties`, and `this.setProperties`.

## Motivation

[RFC #785](https://github.com/emberjs/rfcs/blob/main/text/0785-remove-set-get-in-tests.md) shipped replacements for these methods. We believe that rendering tests should be using the replacements: `render` accepting a component, and `rerender`, landed in `@ember/test-helpers` 2.8.0, backed by `renderSettled` from `@ember/renderer` in `ember-source` 4.5.0.

## Transition Path


### Before

```js
import { render } from '@ember/test-helpers';
import { hbs } from 'ember-cli-htmlbars';

test('it renders the name', async function (assert) {
  this.set('name', 'Zoey');

  await render(hbs`<MyComponent @name={{this.name}} />`);

  assert.dom('[data-test-name]').hasText('Zoey');

  this.set('name', 'Tomster');

  assert.dom('[data-test-name]').hasText('Tomster');
});
```

### After

```js
import { render, rerender } from '@ember/test-helpers';
import { trackedObject } from '@ember/reactive/collections';
import MyComponent from 'my-app/components/my-component';

test('it renders the name', async function (assert) {
  const state = trackedObject({
    name: 'Zoey',
  });

  await render(<template><MyComponent @name={{state.name}} /></template>);

  assert.dom('[data-test-name]').hasText('Zoey');

  state.name = 'Tomster';
  await rerender();

  assert.dom('[data-test-name]').hasText('Tomster');
});
```

### After, for a single value

Many rendering tests hold one value:

```js
import { render, rerender } from '@ember/test-helpers';
import { tracked } from '@glimmer/tracking';
import MyComponent from 'my-app/components/my-component';

test('it renders the name', async function (assert) {
  const name = tracked('Zoey');

  await render(<template><MyComponent @name={{name.value}} /></template>);

  assert.dom('[data-test-name]').hasText('Zoey');

  name.value = 'Tomster';
  await rerender();

  assert.dom('[data-test-name]').hasText('Tomster');
});
```

### `settled()` vs `renderSettled()` vs `rerender()`

- `settled()`, from `@ember/test-helpers`, waits on everything the test framework tracks: rendering, timers, requests, transitions, and test waiters.
- `renderSettled()`, from `@ember/renderer`, waits only for tracked state consumed by the template to reach the DOM.
- `rerender()`, from `@ember/test-helpers`, is a re-export of `renderSettled()`. No behavioral difference.

Use `rerender()` after changing tracked state, `settled()` when the assertion depends on more than rendering, and both when the intermediate state is the point:

```js
state.isLoading = true;
await rerender();
assert.dom('[data-test-status]').hasText('Loading');

await finishLoadingRequest();
await settled();
assert.dom('[data-test-status]').hasText('Loaded');
```

### Ecosystem

- An `eslint-plugin-ember` rule flagging the four methods in tests, so they are caught at lint time rather than at runtime.
- Blueprints in `ember-source` and `ember-cli` that still emit `this.set`.
- A codemod for the mechanical case: state set before `render`, never reassigned. Tests that reassign need `rerender()` placed correctly, and the codemod should skip what it cannot do safely.
- `ember-source` and the addon ecosystem need migration passes of their own.

## How We Teach This

The guides should cover `rerender` versus `settled`, since `this.set`'s flush is what hid the distinction. A missing `await rerender()` will be the most common migration bug, and it presents as an assertion against stale DOM.

API docs mark the four methods deprecated. The deprecation guide covers the before and after in both forms, and states that `this.owner`, `this.element`, `this.pauseTest`, and `this.resumeTest` are unaffected.

## Drawbacks

Mature codebases have thousands of these calls, and the codemod only covers tests that never reassign state, which are the least interesting ones.

The replacement is more verbose for a small test, at least until `tracked()` for a single value ships. A backing class for one field is four lines where `this.set` was one.

Deprecating `get` and `getProperties` is arguably scope creep, since they cause no rendering problems on their own.

## Alternatives

- Do nothing, and leave every `this.set` for whoever lands the render-aware scheduler.
- Deprecate everything except `this.owner`. Considered first on the PR, but `pauseTest` and `resumeTest` are load-bearing for debugging, and stashing properties on `this` is a common pattern.
- Deprecate only `set` and `setProperties`, the two that are `run()`-wrapped. Rejected above.
- Remove without a deprecation period. Not proportionate to how widespread the pattern is.

## Unresolved questions

None.
