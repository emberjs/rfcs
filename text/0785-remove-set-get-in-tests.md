---
Stage: Accepted
Start Date: 2021-11-30
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember
RFC PR: https://github.com/emberjs/rfcs/pull/785
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

# Introduce new test helpers for rendering (and re-rendering) that obviate the need for `get` and `set`

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

On the first point: `this.set` is essentially irrelevant everywhere else in Ember. In a post-Octane world, we no longer need to `get` or `set`, and it is, in fact, frequently discouraged as an unnecessary complication. Additonally, the value of `this` inside of a test is extremely murky. The fact that `this` is both the test context _and_ a sort of ersatz scope for the surrounding "template" of the rendering test is not exactly intuitive or easy to teach.

On the second point: `this.set` is actually [run-wrapped when called in tests](https://github.com/emberjs/ember-test-helpers/blob/master/addon-test-support/@ember/test-helpers/setup-context.ts#L412-L423), which means that every time someone calls `this.set`, they are actually synchronously flushing the entirety of the DOM and triggering a re-render of the entire test template. This _in no way_ resembles the relationship between data updates and re-renders in application code. In application code, changes to the underlying data in template are coalesced into a single DOM update, which, in turn, re-renders only the parts of the DOM that are affected by the changed data. As a result, rendering tests end up essentially testing a version of rendering that isn't even used in the application being tested.

On the third point: having to set properties on `this` leads to a whole bunch of papercuts when writing tests in TypeScript. [As discussed in the `ember-cli-typescript` docs](https://docs.ember-cli-typescript.com/ember/testing#the-testcontext), rendering tests require developers to arbitrarily add (and change) properties on `this`. Developers thus have to define (and re-define) the interface for `this` to accommodate _all_ of those properties to prevent TypeScript from complaining that each of the values that were set don't exist on the `TestContext` type. Beyond simply being annoying, this actually makes it impossible to write safe and useful types spanning multiple tests: properties declared for one test appear to be available in all of the tests in a module (and get autocompleted accordingly), whether or not they actually _are_ available.

The goal of this RFC, then, is to greatly simplify the testing model by removing the need for `get` and `set` in rendering tests, thereby reducing the reliance on `this`.

## Detailed design

We will introduce two new/refactored utilities for use in rendering tests:

1. Update the `render` helper to accept either a component or a template snippet.
1. A new `rerender` function that would be used after the initial `render` call to wait for DOM updates

These additions, used in concert with [First Class Component Templates](https://github.com/emberjs/rfcs/blob/master/text/0779-first-class-component-templates.md), should significantly improve the DX of writing component tests in Ember.

### Updated `render` helper

Rendering tests currently work by rendering a template that is bound to the `this` of the surrounding test. This is why developers have to call `this.set` in the first place, since the value of `this` in a given test is doing double duty as both the actual test context AND the backing object for the template that eventually gets rendered. Since we want to move away from this very behavior, we need a version of `render` that can alternatively accept a fully-formed component, since components have their own context and are self-contained.

The updated version of `render` would be imported from `@ember/test-helpers` and have the following signature:

```ts
function render(template: TemplateFactory | Component, options?: RenderOptions): Promise<void>;
```

Since this new version of `render` will need to differentiate between components and templates, we'll also add an `isComponent` utility to the `@glimmer/runtime` package. This addition will significantly reduce the amount of private API that would be required when implementing the new `render` function.

Additionally, since passing a component to `render` also precludes the user from being able to access the text context via `this` (since components will use their own local context rather than the test's), we'll display a warning message if someone passes a component to `render` while also using `this.set` in the same test. This will help avoid confusion in the case that they opt in to the new component-based testing model but then copy an example from the guides or an older writeup that uses `this.set`.

### `rerender`

Finally, we need to handle cases where you want to update the backing data and assert against the resulting changes in the rendered output. In the current testing paradigm, we'd _usually_ just update the values using `this.set`, which immediately triggers a rerender. In more complex cases, e.g. where there is some async operation involved, we would wait for the state we want to assert against, likely using either `settled` or `waitFor`. This works, but has the downside of either waiting for _everything_ to finish (in the case of `settled`), or waiting for a single thing to finish (in the case of `waitFor`).

Instead, we propose adding a new `rerender` function to `@ember/test-helpers` that exclusively waits on pending render operations, but ignores all other [settledness metrics](https://github.com/emberjs/ember-test-helpers/blob/master/API.md#issettled):

```ts
rerender(): Promise<void>
```

In order to implement `rerender`, we would also expose the work done by @rwjblue on [`renderSettled`](https://github.com/emberjs/ember.js/blob/703b9ca2653a6b479a762b30dca1a33eaa13d8ab/packages/%40ember/-internals/glimmer/lib/renderer.ts#L219-L240) as public API in a new `@ember/renderer` module so that it could then be imported and used by `@ember/test-helpers` without issuue.

When combined with [tracked state](https://guides.emberjs.com/release/in-depth-topics/autotracking-in-depth/#toc_autotracking-basics), this new `rerender` function allows developers to make updates to their component state and `await` the newly rendered version of their component _without_ having to also wait for other pending timers or run loops. In other words, they can still assert against changes in the DOM, but would also be able to assert against things like loading states without having to use `waitFor` since `rerender` would not wait for things like async operations to complete.

Continuing the example from above, here's what we'd expect an update and re-render to look like:

```js
import { render, rerender } from '@ember/test-helpers';
import { tracked } from '@glimmer/tracking';

// setup elided
test('it works', async function (assert) {
  const somePerson = new class {
    @tracked name = 'Zoey',
    @tracked age = 5,
  };

  const component = <template>
    <ProfileCard @name={{somePerson.name}} @age={{somePerson.age}} />
  </template>;

  await render(component);

  assert.dom(this.element).hasText(somePerson.name);

  somePerson.name = 'Zoeyyyyyyyyyyyyy';

  await rerender();

  assert.dom(this.element).hasText(somePerson.name);
});
```

It's worth noting that rendering tests without `get`/`set` are actually possible today using a combination of the `precompileTemplate` function from `@ember/template-compilation`, a scope object containing all of the relevant values referenced in the template, and the `settled` function from `@ember/test-helpers`. Accordingly, the implementation of this RFC would largely involve exposing these existing APIs in a way that is both more user-friendly and better-suited to both [First-Class Component Templates](https://github.com/emberjs/rfcs/pull/779) and [TypeScript](https://github.com/emberjs/rfcs/pull/724), rather than introducing a significant amount of new internal functionality.

## How we teach this

We'd need to add a new TypeScript-specific section to the [testing components](https://guides.emberjs.com/release/testing/testing-components/) section of the official guides to document this new approach for TypeScript users.

We expect there will be a future RFC that introduces a form which also allows passing arguments into the invoking component, in addition to the approach described in this RFC that requires a backing class in order to mark primitives as tracked. As a result, for the time being, we'll only teach this new approach in a TypeScript-specific subsection of the guides on testing components until that followup RFC has been written and merged. We believe that the benefits to the TypeScript community (namely, no longer needing to fight with the type checker over the type of `this` in each test) still significantly outweigh the costs of the slightly awkward method of passing arguments.

We'd also need to update the [API docs for @ember/test-helpers](https://github.com/emberjs/ember-test-helpers/blob/master/API.md) for the new version of `render` as well as the newly introduced `rerender` function.

API documentation for the two new functions are included below:

### render

```ts
/**
  Renders the provided template or component and appends it to the DOM, overwriting any previously-rendered template or component.
  @public
  @param {CompiledTemplate|Component} template the template or component to render
  @param {RenderOptions} options options hash containing engine owner ({ owner: engineOwner })
  @returns {Promise<void>} resolves when settled
*/
render(template: CompiledTemplate | Component, options?: RenderOptions): Promise<void>;
```

### rerender

```ts
/**
  Returns a promise that resolves when rendering has settled.  Settled in this context is defined as when all of the
  tags in use are "current". When this is checked at the _end_ of the run loop, this essentially guarantees that all
  rendering is completed.
  @public
  @returns {Promise<void>} resolves when settled
*/
rerender(): Promise<void>
```

All things considered, the changes in this RFC should result in a component testing model that is ultimately _more_ intuitive. We could also introduce a configuration option to `@ember/test-helpers` that would allow users to turn on an assertion that prevents rendering tests from being written using the old method to help codebases migrate.

## Drawbacks

Ember's testing setup is one its most valuable features, and this RFC would introduce changes to a testing model that already works fine. These changes would be introduced alongside the existing rendering test models, so they would not constitute a breaking change, but would introduce additional overhead in terms of "old vs. new", similar to the overhead of Octane, or the Grand Testing Unification RFCs.

## Alternatives

- Leave things as they are.
- Encourage people to use `precompileTemplate` along with the current version of `render`.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
> TBD?
