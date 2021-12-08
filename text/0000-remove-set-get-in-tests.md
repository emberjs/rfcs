---
Stage: Accepted
Start Date: 2021-11-30
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember
RFC PR:
---

<!---
Directions for above:

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# <RFC title>

## Summary

Introduce new testing utilities that remove the need for the use of `this.get`/`this.set` in test contexts. This will make rendering tests behave more like application code, and also greatly simplify the process of writing rendering tests in TypeScript since it will remove the need to append type information to `this` as properties are set.

## Motivation

Current rendering tests require developers to set values on `this` in order for them to actually be present in the template that gets rendered as part of the test. For example, if we're testing that a component correctly displays a value that gets passed in as an argument, we'd do something like:

```js
this.set('name', 'foo');

await render(hbs`<MyComponent @name={{this.name}} />`);

assert.dom('[data-test-name]').hasText(this.get('name'));

this.set('name', 'FOO');

assert.dom('[data-test-name]').hasText(this.get('name'));
```

This approach has three main issues:

1. It's not consistent with any other programming model in Ember
1. It synchronously flushes the DOM
1. It makes writing tests in TypeScript a huge pain

On the first point: `this.set` is essentially irrelevant everwhere else in Ember. In a post-Octane world, we no longer need to `get` or `set`, and it is, in fact, frequently discouraged as an unnecessary complication. Additonally, the value of `this` inside of a test is extremely murky. The fact that `this` is both the test context _and_ a sort of ersatz scope for the surrounding "template" of the rendering test is not exactly intuitive or easy to teach.

On the second point: `this.set` is actually [run-wrapped when called in tests](https://github.com/emberjs/ember-test-helpers/blob/master/addon-test-support/@ember/test-helpers/setup-context.ts#L412-L423), which means that every time someone calls `this.set`, they are actually synchronously flushing the entirety of the DOM and triggering a re-render of the entire test template. This _in no way_ resembles the relationship between data updates and re-renders in application code. In application code, changes to the underlying data in template are coalesced into a single DOM update, which, in turn, re-renders only the parts of the DOM that are affected by the changed data. As a result, rendering tests end up essentially testing a version of rendering that isn't even used in the application being tested.

On the third point: having to set properties on `this` leads to a whole bunch of papercuts when writing tests in TypeScript. [As discussed in the `ember-cli-typescript` docs](https://docs.ember-cli-typescript.com/ember/testing#the-testcontext), rendering tests require developers to arbitrarily add (and change) properties on `this`. Developers thus have to define (and re-define) the interface for `this` to accommodate _all_ of those properties to prevent TypeScript from complaining that each of the values that were set don't exist on the `TestContext` type. Beyond simply being annoying, this actually makes it impossible to write safe and useful types spanning multiple tests: properties declared for one test appear to be available in all of the tests in a module (and get autocompleted accordingly), whether or not they actually _are_ available.

The goal of this RFC, then, is to greatly simplify the testing model by removing the need for `get` and `set` in rendering tests, thereby reducing the reliance on `this`.

## Detailed design

We will introduce several new/refactored utilities for use in rendering tests:

1. Update the `render` helper to accept either a component or a template snippet.
1. A new `scopedTemplate` function that accepts a template and returns a component _that uses lexically scoped data_
1. A new `rerender` function that would be used after the initial `render` call to wait for DOM updates

Rendering tests currently work by rendering a template that is bound to the `this` of the surrounding test. This is why developers have to call `this.set` in the first place, since the value of `this` in a given test is doing double duty as both the actual test context AND the backing object for the template that eventually gets rendered. Since we want to move away from this very behavior, we need a version of `render` that accepts a fully-formed component instead, since components have their own context and are self-contained.

Once we have this new version of `render`, we also need a convenient way of creating components to pass to it. This is where the `scopedTemplate` function comes in. `scopedTemplate` would accept a string and return a template-only component which could then be passed to our new version of `render`. (A proposal like `<template>` from [RFC #0779](https://github.com/chriskrycho/ember-rfcs/blob/fcct/text/0779-first-class-component-templates.md) could supersede this, as the components created with first-class component templates could be passed directly to `render` as well. This RFC can move forward and use lexical scoping _without_ requiring that full feature set to be implemented, however.) In addition, the template passed to `scopedTemplate` would have access to lexically scoped data, meaning that its values would pull _directly from the surrounding scope_, i.e. the values defined in the test itself. With this new approach, instead of having to call `this.set` for each value you want to reference in the template, you instead just define the data you want and reference it directly in the template. For example:

```js
import { render } from '@ember/test-helpers';
import { scopedTemplate } from 'ember-cli-htmlbars';

// setup elided
test('it works', async function (assert) {
  const somePerson = {
    name: 'Zoey',
    age: 5,
  };

  const component = scopedTemplate(`
    <ProfileCard @name={{somePerson.name}} @age={{somePerson.age}} />
  `);

  await render(component);

  assert.dom(this.element).hasText(somePerson.name);
});
```

Finally, we need to handle cases where you want to update the backing data and assert against the resulting changes in the rendered output. In the current testing paradigm, we'd _usually_ just update the values using `this.set`, which immediately triggers a rerender. In more complex cases, e.g. where there is some async operation involved, we would wait for the state we want to assert against, likely using either `settled` or `waitFor`. This works, but has the downside of either waiting for _everything_ to finish (in the case of `settled`), or waiting for a single thing to finish (in the case of `waitFor`).

Instead, we propose a new `rerender` function that exclusively waits on pending render operations, but ignores all other [settledness metrics](https://github.com/emberjs/ember-test-helpers/blob/master/API.md#issettled). When combined with [tracked state](https://guides.emberjs.com/release/in-depth-topics/autotracking-in-depth/#toc_autotracking-basics), this new `rerender` function allows developers to make updates to their component state and `await` the newly rendered version of their component _without_ having to also wait for other pending timers or run loops. In other words, they can still assert against changes in the DOM, but would also be able to assert against things like loading states without having to use `waitFor` since `rerender` would not wait for things like async operations to complete.

Continuing the example from above, here's what we'd expect an update and re-render to look like:

```js
import { render, rerender } from '@ember/test-helpers';
import { scopedTemplate } from 'ember-cli-htmlbars';
import { tracked } from '@glimmer/tracking';

// setup elided
test('it works', async function (assert) {
  const somePerson = new class {
    @tracked name = 'Zoey',
    @tracked age = 5,
  };

  const component = scopedTemplate(`
    <ProfileCard @name={{somePerson.name}} @age={{somePerson.age}} />
  `);

  await render(component);

  assert.dom(this.element).hasText(somePerson.name);

  somePerson.name = 'Zoeyyyyyyyyyyyyy';

  await rerender();

  assert.dom(this.element).hasText(somePerson.name);
});
```

It's worth noting that rendering tests without `get`/`set` are actually possible today using a combination of the `precompileTemplate` function from `@ember/template-compilation`, a scope object containing all of the relevant values referenced in the template, and the `settled` function from `@ember/test-helpers`. Accordingly, the implementation of this RFC would largely involve exposing these existing APIs in a way that is both more user-friendly and better-suited to both [First-Class Component Templates](https://github.com/emberjs/rfcs/pull/779) and [TypeScript](https://github.com/emberjs/rfcs/pull/724), rather than introducing a significant amount of new internal functionality.

## How we teach this

We'd need to update the [testing components](https://guides.emberjs.com/release/testing/testing-components/) section of the official guides to document this new approach.

We'd also need to update the [API docs for ember-test-helpers](https://github.com/emberjs/ember-test-helpers/blob/master/API.md) for the new version of `render` as well as the newly introduced `rerender` and `scopedTemplate` functions.

All things considered, the changes in this RFC should result in a component testing model that is ultimately _more_ intuitive. We could also introduce a configuration option to `@ember/test-helpers` that would allow users to turn on assertion that prevents rendering tests from being written using the old method to help codebases migrate.

## Drawbacks

Ember's testing setup is one its most valuable features, and this RFC would introduce changes to a testing model that already works fine. These changes would be introduced alongside the existing rendering test models, so they would not constitute a breaking change, but would introduce additional overhead in terms of "old vs. new", similar to the overhead of Octane, or the Grand Testing Unification RFCs.

## Alternatives

- Leave things as they are.
- Encourage people to use `precompileTemplate` along with the current version of `render`.
- Only implement the changes to `render` and wait for first-class component templates rather than introducing `scopedTemplate`
  - That said, the upside to implementing `scopedTemplate` along with the changes to `render` is that it would use the same primitives as first-class component templates, which allows tests written using `scopedTemplate` to be codemodded easily and safely, and allows us to moving the state of rendering tests forward without having to wait on the much larger first-class component template implementation. In addition, implementing `scopedTemplate` sooner provides a lot of value to TypeScript users immediately.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
> TBD?
