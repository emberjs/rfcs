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

Give Ember a way to contain a render error instead of letting it take down the page. A boundary catches synchronous errors thrown during Glimmer VM render, on both initial render and rerender, shows fallback content in their place, and can recover afterwards. The proposed surface is a built-in `<ErrorBoundary>` component, with a `<:try>` block for the happy path and a `<:catch>` block for the fallback. Recovery comes from the reactivity system: a boundary retries when tracked state read by the failed render changes, and the fallback also gets a `retry` function for errors tracking can't see. The component is the shape suggested here, not the point of the proposal. [Alternatives](#alternatives) compares it with a block keyword, and the choice between them is still open.

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
    <:try>
      <RiskyComponent />
    </:try>
    <:catch as |error retry|>
      <p>Something went wrong: {{error.message}}</p>
      <button {{on "click" retry}}>Retry</button>
    </:catch>
  </ErrorBoundary>
</template>
```

- `<:try>` is rendered when there's no error. This is the "happy path" content.
- `<:catch as |error retry|>` is rendered when an error is caught while rendering `<:try>`.
  - `error` (`unknown`): the caught error object. You'll want to narrow the type before accessing properties.
  - `retry` (`() => void`): a function that clears the error state and renders `<:try>` again. If the underlying issue isn't resolved, the error will be caught again.

The blocks can appear in either order, so a template can put `<:catch>` first when that reads better.

When there's no fallback to show, the content can be passed without named blocks:

```gjs
<ErrorBoundary>
  <RiskyComponent />
</ErrorBoundary>
```

In that form the boundary renders nothing while it is in error state. Glimmer doesn't allow implicit content next to named blocks, so once a `<:catch>` block is present, the happy path has to be named too. It must be `<:try>`. Development builds would assert when an explicit `<:default>` block is combined with `<:try>` or `<:catch>`, so each shape has exactly one spelling.

### Automatic retry

A boundary in error state would retry on its own when tracked state read by the failed render changes. There's no argument to configure. A retry is just a re-render. The reactivity system already records what a render reads, and the boundary would keep that record for the failed render, the same way any other part of a template depends on what it read.

```gjs
<ErrorBoundary>
  <:try>
    <UserCard @user={{this.user}} />  {{! throws while @user is incomplete }}
  </:try>
  <:catch>
    <p>Couldn't show this user.</p>
  </:catch>
</ErrorBoundary>
```

If `UserCard` throws while reading `@user`, setting `this.user` to something valid re-renders `<:try>`, and the fallback goes away without any user interaction. If the new value still makes it throw, the error is caught again.

This is fine-grained:

- **State read before the throw** can retry the boundary. A change to it is the only kind of change that could make the render succeed.
- **State the failed render never read** can't. Changing it may update the `<:catch>` block, but `<:try>` isn't attempted again. If something wasn't read before the error, there's no reason to believe changing it will fix the error.

Route-level recovery falls out of this. A boundary wrapping `{{outlet}}` reads the outlet state before the routed template throws, so navigating to another route retries it:

```gjs
<ErrorBoundary>
  <:try>
    {{outlet}}
  </:try>
  <:catch as |error|>
    <p>This page encountered an error.</p>
  </:catch>
</ErrorBoundary>
```

`retry` remains for failures that tracking can't see: state kept in a plain object, a value read from a global, or a transient failure like a service outage. Nothing tells the boundary that such a failure is over, so a manual retry is the way back.

This shouldn't add cost while the boundary isn't in error state. It only needs to remember the failed render's dependencies while it is showing the fallback.

### Internal template

The component's own template could stay minimal, for example:

```hbs
{{#if this.hasError}}
  {{yield this.error this.retry to="catch"}}
{{else if (has-block "try")}}
  {{yield to="try"}}
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

- Modifier install/update errors: modifiers run in `transaction.commit()`, after the VM execution phase the boundary covers has already finished
- Async errors: errors in `setTimeout`, `requestAnimationFrame`, Promise rejections, `ember-concurrency` tasks, etc.
- Errors during component destruction: destructor callbacks run outside the render pass
- Errors in event handlers: `{{on "click" this.handleClick}}` errors aren't render errors

The boundary's scope is the synchronous Glimmer VM execution pass. Async error handling is a separate concern, better served by application-level patterns.

Modifiers deserve a longer answer, because an uncaught modifier error is just as disruptive as an uncaught render error, and the boundary as proposed would not help. Two things make them harder than render errors rather than merely out of scope:

1. **They run outside the window.** Modifiers are invoked during `transaction.commit()`, once the VM pass the boundary wraps has already completed. Catching them means extending the boundary into the commit phase, which is a separate change to the transaction lifecycle.
2. **Recovery does not mean the same thing.** Recovering from a render error means discarding the DOM the failed render produced. A modifier's whole purpose is to reach outside that model: it may already have mutated the element, attached listeners, or started external work. Removing the element does not undo those effects, so a boundary that "recovered" from a modifier error could leave the app in a worse state than one that let the error surface.

Making this work would need a defined answer for what unwinding a partially applied modifier means, not just a wider `try`. That is left to a follow-up rather than assumed solvable, and it is the most significant known gap in this proposal.

### Nesting

ErrorBoundaries can be nested. When an error occurs, the **innermost** enclosing boundary catches it:

```gjs
<ErrorBoundary>
  <:try>
    <ErrorBoundary>
      <:try>
        <ThrowingComponent /> {{! caught by inner boundary }}
      </:try>
      <:catch as |error|>
        <p>Inner caught: {{error.message}}</p>
      </:catch>
    </ErrorBoundary>
  </:try>
  <:catch as |error|>
    <p>Outer caught: {{error.message}}</p>
  </:catch>
</ErrorBoundary>
```

If the `<:catch>` block itself throws, the error bubbles to the next enclosing boundary (or goes uncaught if there's no parent boundary).

### Error recovery and DOM cleanup

When an error is caught, the boundary would:

1. Roll back the VM's updating opcode list to the state before the failed render
2. Remove any DOM nodes created during the failed render pass
3. Roll back the debug render tree (`debugRenderTree.rollbackTo()`)
4. Unwind the tracking frames left open by the failed render, keeping the tags they consumed as the record of what the failed render read, rather than discarding them
5. Transition to error state and render the `<:catch>` block

While in error state, the boundary would consume the kept tags in its own tracking frame, so a change to any of them re-runs it. When that happens, or when `retry()` is called, the boundary would:

1. Clear the error state
2. Render the `<:try>` block from scratch
3. If it throws again, catch the error again and return to error state, with a fresh record of what was read

One detail matters for the record to be complete, and it is a change to how tracking behaves today. Currently, when a tracked computation throws, the tags it read before the throw are dropped along with its frame. They would need to reach the parent frame instead. Otherwise the boundary would never see the state that caused the error, which is the one it most needs.

### Development mode behavior

In development builds (`DEBUG` is true), ErrorBoundary would log caught errors to the console, for example:

```
console.error('An error was caught by <ErrorBoundary>:', error)
```

This way developers would still be aware of caught errors during development, even though the UI recovers gracefully.

### Ecosystem implications

**Linting:** ErrorBoundary uses standard named blocks, so existing tooling understands it without changes. The block-combination check described in [Basic usage](#basic-usage) is proposed as a development-time assertion, which would fire when a boundary renders. A rule in eslint-plugin-ember could report the same mistakes statically, by following the `ErrorBoundary` identifier back to its `@ember/component` import. The template compiler can't do this reliably on its own, because in strict mode it only sees an identifier, which could be renamed or shadowed.

**Ember Inspector:** The implementation would maintain the debug render tree during error recovery. `debugRenderTree.rollbackTo()` would keep the Inspector's view of the component tree consistent after an error is caught.

**Server-side rendering (FastBoot):** ErrorBoundary would operate at the Glimmer VM level and catch synchronous render errors. It should work in FastBoot without modification since FastBoot uses the same Glimmer VM for rendering. This should be verified in practice.

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
2. Basic usage: wrapping a section of UI in `<ErrorBoundary>` with `<:try>` and `<:catch>` blocks
3. Automatic retry: fixing the state that caused the error recovers the boundary, including navigating away from a broken route
4. The `retry` function: recovering from failures caused by state that isn't tracked
5. What is and isn't caught: clearly explaining the synchronous render scope, and directing developers to other patterns for async errors
6. Nesting boundaries: using multiple boundaries to isolate different sections of the page

### API documentation

The `@ember/component` module documentation should include:

- `ErrorBoundary`: the component class (import path and usage)
- `<:try>` and `<:catch as |error retry|>`: the named blocks and their parameters
- When a boundary retries on its own, and when `retry` is needed

### Teaching approach

ErrorBoundary should be presented as a **progressive enhancement** tool:

- "Identify sections of your UI that could fail independently, and wrap them in `<ErrorBoundary>`"
- Present automatic retry as normal reactivity rather than a separate feature: the boundary re-renders when what it depends on changes, like everything else
- Introduce `retry` as the escape hatch for state tracking can't observe
- Emphasize the limitations clearly: ErrorBoundary is for render errors only, not a general-purpose error handling mechanism

## Drawbacks

### False sense of safety

Developers may assume that wrapping content in `<ErrorBoundary>` catches all possible errors. In practice, modifier errors, async errors, and event handler errors all escape the boundary. This needs to be documented clearly and taught explicitly to avoid a false sense of security.

### VM complexity

An implementation would touch sensitive internal parts of the Glimmer VM: state restoration during error recovery, DOM cleanup of partially-rendered trees, and tracking system integration. This increases the maintenance surface area for the VM team and introduces new code paths that need to be considered during future VM changes.

### Performance

Catching render errors adds work to rendering. No benchmarks have been run yet, so the cost of this proposal is still unknown and would need to be measured.

### Retry churn

Automatic retry follows whatever the failed render read. If that includes state that changes often, like a clock or a value updated on every animation frame, the boundary would attempt `<:try>` on each change, and in development each failed attempt would log the error again. That is arguably correct, since any of those changes could have fixed the error, but it can be surprising. A render that writes to tracked state it already read is a separate problem: it would retry itself in a loop. Ember's existing development-mode assertion against updating state already read in the same render should catch it before it ships.

### Swallowed errors

In production, caught errors aren't surfaced to the user beyond the fallback UI. `console.error` would be emitted in development builds, but production builds would catch errors silently. If ErrorBoundary is overused, especially without logging, it could mask bugs that should be fixed. Best practices should recommend pairing ErrorBoundary with error reporting (e.g., sending caught errors to a monitoring service).

## Alternatives

### Prior discussion

[RFC issue #513](https://github.com/emberjs/rfcs/issues/513) (2019) and [RFC issue #518](https://github.com/emberjs/rfcs/issues/518) (2019) both requested error boundaries for Ember but were never formalized into an RFC. At the VM level, [glimmer-vm#1462](https://github.com/glimmerjs/glimmer-vm/pull/1462) and its follow-up [glimmer-vm#1501](https://github.com/glimmerjs/glimmer-vm/pull/1501) explored error recovery inside the VM, without settling on a user-facing API. There's no known community addon that provides render-level error boundaries, likely because the feature requires changes to the Glimmer VM itself. Existing error handling addons operate at the application level (via `Ember.onerror` / `window.onerror`), not within the component tree.

### React's class-based API

React implements error boundaries via class component lifecycle methods (`componentDidCatch`, `getDerivedStateFromError`). This RFC proposes named blocks instead, which:

- Is more declarative and template-centric
- Doesn't require a class component. ErrorBoundary would work in any template context
- Provides the `retry` function directly as a block parameter, so recovery doesn't require extra wiring

### Block syntax (`{{#try}}` / `{{catch}}`)

The obvious alternative shape is a block form rather than a component:

```hbs
{{#try}}
  <RiskyComponent />
{{catch as |error retry|}}
  <p>Something went wrong: {{error.message}}</p>
{{/try}}
```

This would be a keyword rather than a component, in line with recent releases that made fundamental behaviour into keywords. A keyword without new syntax sigils needs little from tooling beyond Glint and ESLint updates, so tooling cost doesn't decide between the two forms. The template compiler would still need to support `{{catch}}` as a second clause that receives `error` and `retry`.

The main argument for the component is that named blocks have no ordering requirement. `{{#try}}` / `{{catch}}` can only be written one way round, while `<:catch>` can come before `<:try>`. Error boundaries are the first of several render states a template may want to express, alongside loading states for async work. Those are state machines, and named blocks represent each state cleanly without the framework imposing an order on them.

The semantics in this RFC carry over to either form unchanged. The choice between them is still open (see [Unresolved questions](#unresolved-questions)).

### Block names: `<:default>` and `<:error>`

An earlier draft used `<:default>` for the happy path and `<:error>` for the fallback. `<:default>` names the mechanism, the block `{{yield}}` renders, rather than the block's role, and it reads awkwardly next to a fallback. `<:try>` says what the block is for, and `<:catch>` pairs with it. Mixing the try/catch metaphor with a state name like `<:error>` would give neither consistency.

Keeping `<:default>` allowed as a synonym for `<:try>` was also considered. It would give every boundary with a fallback two spellings, so this proposal rejects it, with a development-mode assertion to enforce that. Implicit content still works on its own, for the common case of a boundary with no fallback.

### An explicit retry key (`@retryWith`)

An earlier draft had a `@retryWith` argument: when its value changed, shallowly compared, a boundary in error state would retry. Its main use was retrying on navigation, with `@retryWith={{this.router.currentRouteName}}`.

It isn't needed. The reactivity system already records what a render reads. With the change described in [Error recovery and DOM cleanup](#error-recovery-and-dom-cleanup), that record would survive a throw, and a retry is just another re-render. Retrying on the state the failed render read is also more precise than any key a developer could choose. A key retries on changes that can't affect the error, and misses ones that can. The navigation case is covered too, since a boundary around `{{outlet}}` reads the outlet state before the routed template throws.

### Dropping `retry`

With automatic retry, a manual `retry` could look redundant. If tracking can't see what caused an error, retrying is a guess. A userland version is also possible: read a tracked counter at the top of `<:try>`, and increment it from the `<:catch>` block.

It is kept for three reasons:

- **Some failures leave nothing to watch.** If `<:try>` throws before reading any tracked state, the failed render records nothing, and the boundary could never retry on its own. That covers bugs triggered by untracked data: plain objects, globals, browser storage, or a third-party library's internal state.
- **The host doesn't always own the code that failed.** In the plugin architectures this RFC is motivated by, the host app wraps code it doesn't control and can't make that code read tracked state. Offering the user a way to try again is the one recovery the host can always provide.
- **The userland version depends on an internal detail.** The counter only works because it is read before the throw. That is exactly how the boundary decides when to retry, and it shouldn't become something developers have to know to build a Retry button.

`retry` has a single, simple meaning, "render `<:try>` again", and costs little to keep.

### Status quo (no built-in boundary)

Doing nothing leaves Ember as one of the few major frameworks without declarative render-error recovery. This is particularly painful for apps with plugin architectures, where third-party code can break the host app's UI. Without error boundaries, developers either accept the risk of full-page failures or implement fragile workarounds.

## Unresolved questions

- **Should ErrorBoundary catch modifier errors?** See [What is NOT caught](#what-is-not-caught) for why this is harder than widening the `try`. It needs a defined meaning for unwinding a partially applied modifier, and is the most likely candidate for a follow-up RFC.

- **Component or keyword?** A spike of the keyword form, benchmarked against the component, would help decide. See [Block syntax](#block-syntax-try--catch) and [Performance](#performance).

- **Should the block-combination check be static?** As proposed, it would be a development-mode assertion that fires when a boundary renders, so a template that never renders would go unreported. A lint rule could catch it earlier. See [Ecosystem implications](#ecosystem-implications).

- **FastBoot and Ember Engines compatibility:** ErrorBoundary should work in FastBoot since it uses the same Glimmer VM for rendering, and within engine component trees. These environments should be tested before the feature is marked as stable.

## Proof of concept

A working prototype exists, and the scope of what it is meant to establish is narrow, so it is worth stating plainly up front.

**What it is for.** It answers one question: can this behaviour be built inside the Glimmer VM at all? A boundary has to unwind a partially completed render, including updating opcodes, the DOM produced so far, the debug render tree, and tracking state, and it was not obvious that this could be done without leaving the VM in a corrupt state. The prototype demonstrates that it can, and it makes the proposed API concrete enough to argue about.

**What it is not.** It is not a proposed implementation, and it is not offered as a pull request against Ember. It was built leaning heavily on AI assistance, which was well suited to exploring an unfamiliar part of the VM quickly but does not substitute for the design judgement of people who maintain it. The code has not been reviewed to the standard Ember would require, and it should be read as evidence that the feature is achievable rather than as a suggestion of how it ought to be written. Should this RFC advance, the implementation should be designed by the framework team, informed by the prototype where useful and discarded where not.

Reviewers are asked to evaluate the proposed API and semantics on their own merits. The prototype is supporting evidence, not the proposal.

- **Implementation**: [megothss/ember.js#2](https://github.com/megothss/ember.js/pull/2), a fork of ember-source carrying the Glimmer VM changes, the ErrorBoundary component, and test coverage
- **Live demo**: [ember-error-boundary-demo](https://megothss.github.io/ember-error-boundary-demo/), a standalone Ember app covering render errors, automatic and manual retry, fine-grained retry, nested boundaries, sibling isolation, route-level recovery, and `{{#in-element}}` portals
