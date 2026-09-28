---
stage: accepted
start-date: 2026-09-09T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - framework
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite:
---

# Error boundaries

## Summary

Give Ember a way to contain a render error instead of letting it take down the page. A boundary catches synchronous errors thrown during Glimmer VM render, on both initial render and rerender, shows fallback content in their place, and can recover afterwards. The proposed surface is a built-in `<ErrorBoundary>` component, using named blocks for the happy path and the fallback, with a `@retryWith` argument for automatic recovery. The component is the shape suggested here, not the point of the proposal, and [Alternatives](#alternatives) covers why it was preferred to a block keyword.

## Motivation

Today, when a component throws during render, the error propagates up uncaught and can leave the page in a broken or unresponsive state. There's no declarative mechanism to catch these errors, display fallback UI, or recover without a full page reload. This is a significant gap in Ember's component model.

```gjs
import Component from '@glimmer/component';

class BuggyWidget extends Component {
  get title() {
    return this.args.data.title; // throws if @data is undefined
  }

  <template>
    <h2>{{this.title}}</h2>
  </template>
}

<template>
  <Header />
  <Sidebar />
  <BuggyWidget />  {{! this error takes down the entire page }}
  <Footer />
</template>
```

In this example, `BuggyWidget` throws because `@data` was not passed. But the failure isn't isolated to the widget. The entire page, including `<Header>`, `<Sidebar>`, and `<Footer>`, is left broken with no way to recover.

### Real-world need: plugin architectures

Large applications with plugin or extension systems allow third-party code to render components within the host app's component tree. A bug in a single plugin can take down the entire page: the sidebar, the header, the content area, everything. ErrorBoundary would let the host app isolate plugin-rendered sections so that a failure in one plugin only affects that plugin's UI.

### Framework parity

Every other major frontend framework provides error boundaries:

- **React**: [`componentDidCatch` / `getDerivedStateFromError`](https://react.dev/reference/react/Component#catching-rendering-errors-with-an-error-boundary) (since v16, 2017)
- **Solid**: [`<ErrorBoundary>`](https://docs.solidjs.com/concepts/control-flow/error-boundary)
- **Vue**: [`onErrorCaptured`](https://vuejs.org/api/composition-api-lifecycle.html#onerrorcaptured)
- **Preact**: [`componentDidCatch`](https://preactjs.com/guide/v10/components/#error-boundaries)
- **Svelte**: [`<svelte:boundary>`](https://svelte.dev/docs/svelte/svelte-boundary) (since v5.3, 2024)

Ember is one of the few remaining major frameworks without declarative render-error recovery.

## Detailed design

### Import

```js
import { ErrorBoundary } from "@ember/component";
```

ErrorBoundary would be a built-in component shipped as part of the framework, not an addon. Available in any Ember app without additional installation.

### Basic usage

ErrorBoundary uses Ember's named blocks syntax:

```gjs
import { ErrorBoundary } from '@ember/component';

<template>
  <ErrorBoundary>
    <:default>
      <RiskyComponent />
    </:default>
    <:error as |error retry|>
      <p>Something went wrong: {{error.message}}</p>
      <button {{on "click" retry}}>Retry</button>
    </:error>
  </ErrorBoundary>
</template>
```

- `<:default>` is rendered when there's no error. This is the "happy path" content.
- `<:error as |error retry|>` is rendered when an error is caught during the default block's render.
  - `error` (`unknown`): the caught error object. You'll want to narrow the type before accessing properties.
  - `retry` (`() => void`): a function that clears the error state and re-renders the default block. If the underlying issue isn't resolved, the error will be caught again.

When used without an explicit `<:default>` block, the implicit default block is used:

```gjs
<ErrorBoundary>
  <RiskyComponent />
</ErrorBoundary>
```

Without an `<:error>` block, caught errors won't have any visible fallback (the boundary will simply render nothing when in error state). In practice, you should always provide both blocks.

### The `@retryWith` argument

```gjs
import { ErrorBoundary } from '@ember/component';
import { service } from '@ember/service';

class MyComponent {
  @service router;

  <template>
    <ErrorBoundary @retryWith={{this.router.currentRouteName}}>
      <:default>
        {{outlet}}
      </:default>
      <:error as |error retry|>
        <p>This page encountered an error.</p>
        <button {{on "click" retry}}>Retry</button>
      </:error>
    </ErrorBoundary>
  </template>
}
```

`@retryWith` accepts any value: a primitive, an array, or a plain object. When the boundary is in error state and the `@retryWith` value changes (determined by shallow equality), the error is automatically cleared and the default block re-renders.

Shallow equality semantics:

- Primitives: compared with `===`
- Arrays: compared element-wise (same length, each element `===`)
- Plain objects: compared by own-property values (same keys, each value `===`)

When the boundary is _not_ in error state, `@retryWith` value changes are tracked but don't trigger any re-render or recovery logic. There's no performance cost in the happy path beyond the tracking overhead.

### Internal template

The ErrorBoundary component's template is minimal:

```hbs
{{#if this.hasError}}
  {{yield this.error this.retry to="error"}}
{{else}}
  {{yield}}
{{/if}}
```

The actual error-catching behavior would be implemented at the Glimmer VM level, not in the template. A new `errorBoundary: true` capability flag on the component manager would signal to the VM that this component should catch errors during its child tree's render.

### What IS caught

It catches synchronous errors that occur during the Glimmer VM's execution phase:

- Component getter/render errors: errors thrown from tracked getters accessed during render, both initial render and rerender
- Helper invocation errors: errors thrown from helpers invoked in templates
- `{{#each}}` list sync errors: errors thrown during iteration key computation or list reconciliation
- Conditional branch transitions: errors thrown when entering an `{{#if}}` / `{{else}}` branch

### What is NOT caught

It doesn't catch:

- Modifier install/update errors: modifiers run in `transaction.commit()`, after the VM execution phase has finished and the boundary's try/catch has already exited
- Async errors: errors in `setTimeout`, `requestAnimationFrame`, Promise rejections, `ember-concurrency` tasks, etc.
- Errors during component destruction: destructor callbacks run outside the render pass
- Errors in event handlers: `{{on "click" this.handleClick}}` errors aren't render errors

The boundary's scope is the synchronous Glimmer VM execution pass. Async error handling is a separate concern, better served by application-level patterns.

Modifiers deserve a longer answer, because an uncaught modifier error is just as disruptive as an uncaught render error, and the current boundary does not help. Two things make them harder than render errors rather than merely out of scope:

1. **They run outside the window.** Modifiers are invoked during `transaction.commit()`, once the VM pass the boundary wraps has already completed. Catching them means extending the boundary into the commit phase, which is a separate change to the transaction lifecycle.
2. **Recovery does not mean the same thing.** Recovering from a render error means discarding the DOM the failed render produced. A modifier's whole purpose is to reach outside that model: it may already have mutated the element, attached listeners, or started external work. Removing the element does not undo those effects, so a boundary that "recovered" from a modifier error could leave the app in a worse state than one that let the error surface.

Making this work would need a defined answer for what unwinding a partially applied modifier means, not just a wider `try`. That is left to a follow-up rather than assumed solvable, and it is the most significant known gap in this proposal.

### Nesting

ErrorBoundaries can be nested. When an error occurs, the **innermost** enclosing boundary catches it:

```gjs
<ErrorBoundary>
  <:default>
    <ErrorBoundary>
      <:default>
        <ThrowingComponent /> {{! caught by inner boundary }}
      </:default>
      <:error as |error|>
        <p>Inner caught: {{error.message}}</p>
      </:error>
    </ErrorBoundary>
  </:default>
  <:error as |error|>
    <p>Outer caught: {{error.message}}</p>
  </:error>
</ErrorBoundary>
```

If the `<:error>` block itself throws, the error bubbles to the next enclosing boundary (or goes uncaught if there's no parent boundary).

### Error recovery and DOM cleanup

When an error is caught:

1. The VM's updating opcode list is rolled back to the state before the failed render
2. Any DOM nodes created during the failed render pass are removed
3. The debug render tree is rolled back (`debugRenderTree.rollbackTo()`)
4. The boundary transitions to error state and renders the `<:error>` block

When `retry()` is called (or `@retryWith` triggers automatic recovery):

1. The error state is cleared
2. The boundary re-renders the `<:default>` block from scratch
3. If the default block throws again, the error is caught again and the boundary returns to error state

### Development mode behavior

In development builds (`DEBUG` is true), ErrorBoundary logs caught errors to the console:

```
console.error('An error was caught by <ErrorBoundary>:', error)
```

This way developers are still aware of caught errors during development, even though the UI recovers gracefully.

### Ecosystem implications

**ember-template-lint:** No new lint rules needed. ErrorBoundary uses standard named blocks syntax, which is already supported.

**Ember Inspector:** The implementation would maintain the debug render tree during error recovery. `debugRenderTree.rollbackTo()` would keep the Inspector's view of the component tree consistent after an error is caught.

**Server-side rendering (FastBoot):** ErrorBoundary operates at the Glimmer VM level and catches synchronous render errors. It should work in FastBoot without modification since FastBoot uses the same Glimmer VM for rendering. This should be verified in practice.

**Ember Engines:** ErrorBoundary should work within a single engine's component tree. This needs real-world verification.

**TypeScript:** ErrorBoundary would be fully typed. The `error` block parameter would be typed as `unknown`, so you'd need to narrow the type before accessing properties, which is standard TypeScript practice.

**Addons:** Addon authors can use ErrorBoundary to make their components more resilient. Host apps can wrap addon-provided components in boundaries to isolate failures.

## How we teach this

### Naming

"ErrorBoundary" is established terminology across the frontend ecosystem. React introduced the concept in 2017, and Solid, Vue, and Preact all use the same or similar naming. Using `ErrorBoundary` in Ember:

- Reduces cognitive overhead for developers coming from other frameworks
- Makes the feature immediately searchable and discoverable
- Aligns with existing community expectations

### Guides

A new section should be added to the Ember guides under "Components":

**"Handling Render Errors with ErrorBoundary"**

The guide should cover:

1. Why you need error boundaries: what happens when a component throws during render, and why you want to isolate failures
2. Basic usage: wrapping a section of UI in `<ErrorBoundary>` with `<:default>` and `<:error>` blocks
3. The retry pattern: using the `retry` function to let users attempt recovery
4. Automatic recovery with `@retryWith`: binding to route name or other reactive values for automatic recovery when context changes
5. What is and isn't caught: clearly explaining the synchronous render scope, and directing developers to other patterns for async errors
6. Nesting boundaries: using multiple boundaries to isolate different sections of the page

### API documentation

The `@ember/component` module documentation should include:

- `ErrorBoundary`: the component class (import path and usage)
- `@retryWith`: the argument for automatic recovery
- `<:default>` and `<:error as |error retry|>`: the named blocks and their parameters

### Teaching approach

ErrorBoundary should be presented as a **progressive enhancement** tool:

- "Identify sections of your UI that could fail independently, and wrap them in `<ErrorBoundary>`"
- Start with the manual `retry` pattern as the primary recovery mechanism
- Introduce `@retryWith` as an advanced pattern for route-level or context-dependent recovery
- Emphasize the limitations clearly: ErrorBoundary is for render errors only, not a general-purpose error handling mechanism

## Drawbacks

### False sense of safety

Developers may assume that wrapping content in `<ErrorBoundary>` catches all possible errors. In practice, modifier errors, async errors, and event handler errors all escape the boundary. This needs to be documented clearly and taught explicitly to avoid a false sense of security.

### VM complexity

The implementation touches sensitive internal parts of the Glimmer VM: state restoration during error recovery, DOM cleanup of partially-rendered trees, and tracking system integration. This increases the maintenance surface area for the VM team and introduces new code paths that need to be considered during future VM changes.

### Swallowed errors

In production, caught errors aren't surfaced to the user beyond the fallback UI. `console.error` is emitted in development builds, but production builds catch errors silently. If ErrorBoundary is overused, especially without logging, it could mask bugs that should be fixed. Best practices should recommend pairing ErrorBoundary with error reporting (e.g., sending caught errors to a monitoring service).

## Alternatives

### Prior discussion

[RFC issue #513](https://github.com/emberjs/rfcs/issues/513) (2019) and [RFC issue #518](https://github.com/emberjs/rfcs/issues/518) (2019) both requested error boundaries for Ember but were never formalized into an RFC. There's no known community addon that provides render-level error boundaries, likely because the feature requires changes to the Glimmer VM itself. Existing error handling addons operate at the application level (via `Ember.onerror` / `window.onerror`), not within the component tree.

### React's class-based API

React implements error boundaries via class component lifecycle methods (`componentDidCatch`, `getDerivedStateFromError`). This RFC proposes named blocks instead, which:

- Is more declarative and template-centric
- Doesn't require a class component. ErrorBoundary works in any template context
- Provides the `retry` function directly as a block parameter, so recovery doesn't require extra wiring

### Block syntax (`{{#try}}` / `{{catch}}`)

The obvious alternative shape is a block form rather than a component:

```hbs
{{#try}}
  <RiskyComponent />
  {{catch error retry}}
  <p>Something went wrong: {{error.message}}</p>
{{/try}}
```

The component was chosen initially for a practical reason rather than an aesthetic one: it requires no tooling change. `<ErrorBoundary>` with `<:default>` and `<:error>` blocks is built entirely from syntax that already exists, so ember-template-lint, the Prettier template plugin, Glint, syntax highlighting and the language server all understand it on the day it ships. Nothing needs to learn a new construct.

A block keyword would not be free. `{{#try}}` would have to be taught to the template compiler, and `{{catch}}` is the harder half, because it has to be a second clause inside the same block and it has to receive `error` and `retry`. The one clause templates have today, `{{else}}`, takes no parameters, so `{{catch}}` could not be built on top of it. It would be genuinely new syntax. Named blocks give us a fallback clause with parameters already.

None of that makes a keyword the wrong answer, and the semantics in this RFC carry over to one unchanged. It is a statement about cost: the component form can be evaluated, and shipped, without a coordinated change across the template toolchain. If the framework team would rather spend that cost for a nicer surface syntax, this proposal does not stand in the way.

### `@key` instead of `@retryWith`

An alternative name `@key` was considered for the automatic retry argument. This was rejected because `@key` in Ember's `{{#each}}` helper represents a property path for identity tracking, not a reactive value for triggering side effects. `@retryWith` communicates the "retry" intent clearly and avoids confusion with existing Ember concepts.

### Status quo (no built-in boundary)

Doing nothing leaves Ember as one of the few major frameworks without declarative render-error recovery. This is particularly painful for apps with plugin architectures, where third-party code can break the host app's UI. Without error boundaries, developers either accept the risk of full-page failures or implement fragile workarounds.

## Unresolved questions

- **Should ErrorBoundary catch modifier errors?** See [What is NOT caught](#what-is-not-caught) for why this is harder than widening the `try`. It needs a defined meaning for unwinding a partially applied modifier, and is the most likely candidate for a follow-up RFC.

- **FastBoot and Ember Engines compatibility:** ErrorBoundary should work in FastBoot since it uses the same Glimmer VM for rendering, and within engine component trees. These environments should be tested before the feature is marked as stable.

## Proof of concept

A working prototype exists, and the scope of what it is meant to establish is narrow, so it is worth stating plainly up front.

**What it is for.** It answers one question: can this behaviour be built inside the Glimmer VM at all? A boundary has to unwind a partially completed render, including updating opcodes, the DOM produced so far, the debug render tree, and tracking state, and it was not obvious that this could be done without leaving the VM in a corrupt state. The prototype demonstrates that it can, and it makes the proposed API concrete enough to argue about.

**What it is not.** It is not a proposed implementation, and it is not offered as a pull request against Ember. It was built leaning heavily on AI assistance, which was well suited to exploring an unfamiliar part of the VM quickly but does not substitute for the design judgement of people who maintain it. The code has not been reviewed to the standard Ember would require, and it should be read as evidence that the feature is achievable rather than as a suggestion of how it ought to be written. Should this RFC advance, the implementation should be designed by the framework team, informed by the prototype where useful and discarded where not.

Reviewers are asked to evaluate the proposed API and semantics on their own merits. The prototype is supporting evidence, not the proposal.

- **Implementation**: [megothss/ember.js#2](https://github.com/megothss/ember.js/pull/2), a fork of ember-source carrying the Glimmer VM changes, the ErrorBoundary component, and test coverage
- **Live demo**: [ember-error-boundary-demo](https://megothss.github.io/ember-error-boundary-demo/), a standalone Ember app covering render errors, retry and recovery, nested boundaries, sibling isolation, `@retryWith`, and `{{#in-element}}` portals
