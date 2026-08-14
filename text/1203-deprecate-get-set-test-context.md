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

[RFC #785](https://github.com/emberjs/rfcs/blob/main/text/0785-remove-set-get-in-tests.md) made the case against these methods five years ago and shipped the replacements: `render` learned to accept a component, and `rerender` was added alongside it, in `@ember/test-helpers` 2.8.0, backed by `renderSettled` from the `@ember/renderer` module that landed in `ember-source` 4.5.0. What that RFC did not do was set an end date for the old way. This one does.

The arguments have not changed since #785, so they are worth restating only briefly. `get` and `set` are not how anyone writes application code after Octane, so tests written this way are teaching a model that exists nowhere else. Stashing template state on `this` means TypeScript users have to widen `TestContext` per module, and those widenings then appear to apply to every test in the module whether or not the property is actually there. And the test context doing double duty, as both test harness *and* backing object for the template, is just hard to explain.

There is one newer argument. `this.set` and `this.setProperties` are `run()`-wrapped:

```js
Object.defineProperty(context, 'setProperties', {
  value(hash) {
    return run(function () {
      return setProperties(context, hash);
    });
  },
  // ...
});
```

Every call synchronously flushes the entire DOM. Nothing in an application behaves this way; there, updates to tracked state coalesce into one render pass. That synchronous flush is also precisely the behavior that a render-aware scheduler ([RFC #957](https://github.com/emberjs/rfcs/pull/957)) cannot preserve. To be clear: this deprecation is not a prerequisite for that work. A test that already avoids these methods can adopt an async scheduler as-is. But every test that still calls `this.set` is a test that will have to be rewritten when the scheduler changes, and it is better to rewrite it against a deprecation with a migration guide than against a scheduler change.

Note that `get` and `getProperties` are not `run()`-wrapped. They are thin wrappers over `get`/`getProperties` from `@ember/object` applied to the context. They are included here because they exist only to read back what `set` wrote, and keeping them after `set` is gone serves no one.

## Transition Path

### What is deprecated

`setupContext` from `@ember/test-helpers` installs `get`, `set`, `getProperties`, and `setProperties` on the context. All four are deprecated. Because they come from `setupContext` and not `setupRenderingContext`, they are present in unit, rendering, and application tests alike, and the deprecation covers all three.

### What is not deprecated

- `this.owner`, which is the whole point of the test context.
- `this.element`, available in rendering tests. Whether it should exist at all is a separate conversation; this RFC does not have it.
- `this.pauseTest()` and `this.resumeTest()`, which are useful precisely because you can reach for them mid-debug without editing your imports.
- Assigning your own properties to `this`. `this.foo = someValue` is a common pattern for sharing setup between hooks and tests and remains supported.

That last point has a consequence worth spelling out, because it is easy to misread this RFC as doing more than it does. `render` installs the test context as the rendered outlet's `controller`, which is what makes `{{this.name}}` in an `hbs` template resolve against the test context. Deprecating these four methods does not remove that binding:

```js
// still works after this deprecation
this.name = 'Zoey';
await render(hbs`{{this.name}}`); // renders "Zoey"

this.name = 'Tomster';
await rerender();
assert.dom().hasText('Zoey'); // ...and still "Zoey"
```

The initial render picks the value up; the reassignment does nothing, because the property is not tracked and nothing is `run()`-wrapping the write. Severing the template-to-context binding is a larger, separate change and needs its own RFC.

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

The `await rerender()` is new, and it is not optional. `this.set` gave you a synchronous flush for free; a plain assignment to tracked state does not.

Projects not yet on `<template>` can get the same result with `precompileTemplate` and a scope hash. Everything referenced in the template has to be in `scope`, including the component:

```js
import { render, rerender } from '@ember/test-helpers';
import { precompileTemplate } from '@ember/template-compilation';
import { tracked } from '@glimmer/tracking';
import MyComponent from 'my-app/components/my-component';

test('it renders the name', async function (assert) {
  class State {
    @tracked name = 'Zoey';
  }

  const state = new State();

  await render(
    precompileTemplate('<MyComponent @name={{state.name}} />', {
      scope: () => ({ state, MyComponent }),
    })
  );

  assert.dom('[data-test-name]').hasText('Zoey');

  state.name = 'Tomster';
  await rerender();

  assert.dom('[data-test-name]').hasText('Tomster');
});
```

### What to `await`

There are two things to wait on, not three, and the difference between them is the only thing worth teaching here.

`rerender()` from `@ember/test-helpers` is `renderSettled()` from `@ember/renderer`. The entire implementation is:

```js
function rerender() {
  return renderSettled();
}
```

Use whichever import you find more convenient. `rerender` is the one most tests already import, since it comes from the same module as `render` and `click`; `renderSettled` is the one to reach for in code that has no business depending on `@ember/test-helpers`, such as a library that needs to flush rendering. There is no behavioral difference to document, and the guides should not imply one.

Both resolve when auto-tracked state consumed by the template has been written to the DOM. Neither waits on timers, pending requests, route transitions, or test waiters.

`settled()` waits for all of that: every registered settledness metric, rendering included.

So: after changing tracked state, `await rerender()`. When the assertion depends on work beyond rendering, `await settled()`. The two compose when you want to catch an intermediate state before the slow thing finishes:

```js
state.isLoading = true;
await rerender();
assert.dom('[data-test-status]').hasText('Loading');

await finishLoadingRequest();
await settled();
assert.dom('[data-test-status]').hasText('Loaded');
```

One implementation detail that surprises people: `render()` itself resolves with `settled()`, not `renderSettled()`. The initial render therefore waits for full settledness regardless of which helper you use afterward.

### Ecosystem

A rule in `eslint-plugin-ember` should flag `this.get`, `this.set`, `this.getProperties`, and `this.setProperties` inside `test()` and hook callbacks. Catching these at lint time rather than at runtime is what makes the migration tractable for the codebases that have the most of them.

Blueprints in `ember-source` and `ember-cli` that still emit `this.set` in generated rendering tests need updating. New apps should not be generating deprecated code on day one.

A codemod can handle the mechanical case: a fixed set of `this.set` calls before `render`, with no reassignment afterward. Anything that reassigns mid-test needs a `rerender()` inserted at the right point, and anything with conditional or looped state changes needs a human. The codemod should skip what it cannot do safely rather than guess.

`ember-source` and the wider addon ecosystem use these methods heavily in their own test suites and will need migration passes of their own. That is where the codemod earns its keep.

## How We Teach This

The [Testing Components](https://guides.emberjs.com/release/testing/testing-components/) guide is the main lift. It currently teaches `this.set` as the way to get data into a rendering test, so it needs rewriting rather than patching: local tracked state, `render` with a component or `<template>`, `rerender` after changes. RFC #785 deliberately confined the new pattern to a TypeScript-specific subsection, on the grounds that it was awkward without a way to pass arguments directly. That caveat has aged out. `<template>` is the default authoring format now, and the pattern is no longer TypeScript-specific. It should be taught as *the* way to write a rendering test.

The guides should also make the `rerender` / `settled` distinction explicit, since `this.set`'s synchronous flush is exactly the crutch that hid it. The most common migration bug will be a missing `await rerender()`, and it presents as an assertion against stale DOM.

`@ember/test-helpers` API docs mark all four methods deprecated, pointing at the deprecation guide.

The deprecation guide entry on `deprecations.emberjs.com` covers the four methods, why they are going away, the before/after pair above in both `<template>` and `precompileTemplate` form, the `rerender` requirement after reassignment, and an explicit note that `this.owner`, `this.element`, `this.pauseTest`, and `this.resumeTest` are unaffected. That last item matters because the first question anyone reading a deprecation about "the test context" will ask is whether their `this.owner.lookup` calls are next.

## Drawbacks

The migration is large. These methods date to the original testing story, and mature codebases have them in the thousands. A codemod covers the simple shape but not the tests that reassign state, which are the ones worth testing in the first place.

The replacement is more verbose for small tests. `this.set('value', x)` is one line; a tracked class plus an instance is four. That cost is real and it is paid in the tests where it buys the least. The honest defense is that the four lines are the same four lines you would write in application code, and that the one-line version was lying about how rendering works.

Deprecating `get` and `getProperties` is arguably scope creep, since they are not `run()`-wrapped and cause none of the rendering problems. Leaving them would mean shipping a read half of an API whose write half is gone.

## Alternatives

Do nothing. The guides keep teaching a model nothing else in Ember uses, and every test that calls `this.set` becomes a problem for whoever lands the render-aware scheduler.

Deprecate everything on the test context except `this.owner`. This was the first shape considered on the PR and it does not survive contact with the details: `this.pauseTest` and `this.resumeTest` are load-bearing for debugging, `this.element` is a separate question, and stashing your own properties on `this` is a legitimate pattern.

Deprecate only `set` and `setProperties`, the two with the rendering problem. Rejected above.

Remove without a deprecation period. Not proportionate to how widespread the pattern is.

## Unresolved questions

None.
