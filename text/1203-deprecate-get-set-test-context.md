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

[RFC #785](https://github.com/emberjs/rfcs/blob/main/text/0785-remove-set-get-in-tests.md) made the case against these methods and shipped the replacements: `render` accepting a component, and `rerender`, in `@ember/test-helpers` 2.8.0, backed by `renderSettled` from `@ember/renderer` in `ember-source` 4.5.0. It did not set an end date for the old way. This one does.

The reasons from #785 still hold. `get` and `set` are not how application code is written after Octane. Storing template state on `this` forces TypeScript users to widen `TestContext` per module, and those widenings then appear to apply to every test in it. And a test context that is both harness and template backing object is hard to teach.

One reason is newer. `set` and `setProperties` are `run()`-wrapped, so each call synchronously flushes the DOM, where an application coalesces updates into a single render pass. That flush is also what a render-aware scheduler ([RFC #957](https://github.com/emberjs/rfcs/pull/957)) cannot preserve. This deprecation is not a prerequisite for that work, since tests avoiding these methods can adopt an async scheduler today. But every remaining `this.set` is a test that will need rewriting when the scheduler lands, and a deprecation with a migration guide is a better place to do that.

`get` and `getProperties` cause none of this. They are included because they exist to read back what `set` wrote.

## Transition Path

`setupContext` from `@ember/test-helpers` installs `get`, `set`, `getProperties`, and `setProperties` on the context. All four are deprecated.

### What is not deprecated

- `this.owner`, which is the whole point of the test context.
- `this.element`, available in rendering tests. Whether it should exist at all is a separate conversation; this RFC does not have it.
- `this.pauseTest()` and `this.resumeTest()`, useful precisely because you can reach for them mid-debug without editing your imports.
- Assigning your own properties to `this`, a common pattern for sharing setup between hooks and tests.

`render` binds the test context as the rendered template's backing object, which is what makes `{{this.name}}` resolve against it. This RFC does not change that binding, so `this.name = 'Zoey'` before `render` still renders "Zoey"; it just will not update on reassignment. Severing the binding needs its own RFC.

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
import { tracked } from '@glimmer/tracking';
import MyComponent from 'my-app/components/my-component';

test('it renders the name', async function (assert) {
  class State {
    @tracked name = 'Zoey';
  }

  const state = new State();

  await render(<template><MyComponent @name={{state.name}} /></template>);

  assert.dom('[data-test-name]').hasText('Zoey');

  state.name = 'Tomster';
  await rerender();

  assert.dom('[data-test-name]').hasText('Tomster');
});
```

The `await rerender()` is not optional. `this.set` gave you a synchronous flush; assigning to tracked state does not.

Projects not yet on `<template>` can use `precompileTemplate` with a scope hash, which has to include the component:

```js
await render(
  precompileTemplate('<MyComponent @name={{state.name}} />', {
    scope: () => ({ state, MyComponent }),
  })
);
```

### What to `await`

`rerender()` from `@ember/test-helpers` is a re-export of `renderSettled()` from `@ember/renderer`. There is no behavioral difference, and the guides should not imply one; use whichever import is convenient. Both resolve once tracked state consumed by the template has reached the DOM, and neither waits on timers, requests, transitions, or test waiters. `settled()` waits for all of that.

So `await rerender()` after changing tracked state, and `await settled()` when the assertion depends on more than rendering. They compose when the intermediate state is the point:

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

[Testing Components](https://guides.emberjs.com/release/testing/testing-components/) teaches `this.set` as the way to get data into a rendering test, so it needs rewriting rather than patching. #785 confined the replacement to a TypeScript-specific subsection because passing arguments was awkward without `<template>`; that is no longer true, and it should now be taught as the default.

The guides should cover `rerender` versus `settled`, since `this.set`'s flush is what hid the distinction. A missing `await rerender()` will be the most common migration bug, and it presents as an assertion against stale DOM.

API docs mark the four methods deprecated. The deprecation guide covers the before and after in both forms, and states that `this.owner`, `this.element`, `this.pauseTest`, and `this.resumeTest` are unaffected.

## Drawbacks

Mature codebases have thousands of these calls, and the codemod only covers tests that never reassign state, which are the least interesting ones.

The replacement is more verbose for small tests: one line becomes four. Those four are the same four you would write in application code, and the one-line version misrepresented how rendering works.

Deprecating `get` and `getProperties` is arguably scope creep. Keeping them means shipping the read half of an API whose write half is gone.

## Alternatives

Do nothing, and every `this.set` becomes a problem for whoever lands the render-aware scheduler.

Deprecate everything except `this.owner`. Considered first on the PR: `pauseTest` and `resumeTest` are load-bearing for debugging, `this.element` is a separate question, and stashing properties on `this` is a common pattern.

Deprecate only `set` and `setProperties`, the two with the rendering problem. Rejected above.

Remove without a deprecation period. Not proportionate to how widespread the pattern is.

## Unresolved questions

None.
