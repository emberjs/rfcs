---
stage: accepted
start-date: 2026-06-10T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1200 
project-link:
suite: 
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
suite: Leave as is
-->

# `makeContext`: Provide, consume 

## Summary

This RFC proposes one new export from `@ember/helper`: `makeContext`, which creates a render-tree-scoped _context_ -- a `Provide` component for binding a value into a region of the render tree, and a `consume` getter for reading the nearest provided value from anywhere below it. This is the Context feature the community has been asking for since (at least) [RFC #975][rfc-975], scoped down to the smallest API that delivers it.

This RFC supersedes [RFC #1154][rfc-1154], incorporating what was learned while implementing it. An implementation of this proposal exists at [emberjs/ember.js#21450][impl-pr].

[rfc-975]: https://github.com/emberjs/rfcs/pull/975
[rfc-1154]: https://github.com/emberjs/rfcs/pull/1154
[rfc-1155]: https://github.com/emberjs/rfcs/pull/1155
[impl-pr]: https://github.com/emberjs/ember.js/pull/21450
[epcc]: https://github.com/customerio/ember-provide-consume-context

## Motivation

Folks want to share state with a _subtree_ of their app without prop-drilling, and without making that state global the way a service is. Today the only way to get real render-tree-scoped state is [ember-provide-consume-context][epcc], which works by [overriding private Glimmer VM classes](https://github.com/customerio/ember-provide-consume-context/blob/def6d34f639d56ebec1c7c8c888f86ec524b8688/ember-provide-consume-context/src/-private/override-glimmer-runtime-classes.ts#L1) -- which creates upgrade risk and a [maintenance burden](https://github.com/customerio/ember-provide-consume-context/pull/49) for both the addon and for ember-source.

[RFC #975][rfc-975] established the motivation and demand for Context in detail (design systems, chart/graph component families, form state, side-by-side trees that each need their own "ambient" state -- go read it, it holds up). Its last open action item was to propose a concrete public API. [RFC #1154][rfc-1154] attempted that with a low-level scope-exploration API (`getScope` / `addToScope`) intended for userland experimentation. Implementing it revealed that the experimentation layer was the wrong thing to make public: exposing render-tree _iteration_ commits us to far more API surface (and far more implementation detail) than exposing the one feature everyone actually wanted to build with it.

So this RFC proposes the feature itself, and nothing else:

- a way to provide a value to a subtree of the render tree,
- a way to consume the nearest provided value from within that subtree,

The expected outcome is that [ember-provide-consume-context][epcc] users (and everyone who wanted Context but wouldn't take on a private-API dependency to get it) have a built-in, supported, typed primitive.

> [!IMPORTANT]
> The APIs proposed here differ from [ember-provide-consume-context][epcc]. This RFC proposes no decorators: `Provide` and `consume` work only synchronously during render.

## Detailed design

### The API

One new export from `@ember/helper` (plus its type):

```ts
import { makeContext, type Context } from '@ember/helper';
// ComponentLike is used for exposition (it's the @glint/template notion of
// "anything invokable as a component") -- the implementation returns an
// internal component class that satisfies this signature.

export function makeContext<T>(): Context<T>;

export interface Context<T> {
  /**
   * A component that makes `@value` available to everything
   * rendered within its block.
   *
   * Renders no DOM of its own -- it only yields.
   */
  Provide: ComponentLike<{
    Args: { value: T };
    Blocks: { default: [] };
  }>;

  /**
   * The value from the nearest enclosing `<Provide>` for this
   * context.
   *
   * Usable in a template -- `{{theme.consume}}` -- or read
   * directly from JS that runs while rendering (e.g. a component
   * constructor or a function helper).
   *
   * Throws if there is no enclosing `<Provide>` for this context,
   * or if read outside of rendering.
   */
  get consume(): T;
}
```

`makeContext` takes **no arguments**. It does not take a default value. The optional type parameter declares the shape of the value; the value itself is supplied at render time via `<Provide @value={{...}}>`.

### Usage

```gts
import { makeContext } from '@ember/helper';

class Theme {
  color = 'dark';
}

const theme = makeContext<Theme>();
const defaultTheme = new Theme();

<template>
  <theme.Provide @value={{defaultTheme}}>

    {{! with let }}
    {{#let theme.consume as |t|}}
      {{t.color}} {{! "dark" }}
    {{/let}}

    {{! passing elsewhere }}
    <SomeCompnoent @foo={{theme.consume}} />
  </theme.Provide>
</template>
```

The context object is shared the same way any other value is shared in strict mode: export it from a module, and import it wherever you provide or consume.

```gjs
// app/theme.js
import { makeContext } from '@ember/helper';

export const theme = makeContext();
```

```gjs
// some deeply nested component
import Component from '@glimmer/component';
import { theme } from 'my-app/theme';

export default class FancyButton extends Component {
  get color() {
    return theme.consume.color;
  }

  <template>
    <button style="color: {{this.color}}">{{yield}}</button>
  </template>
}
```

### Semantics

**Identity.** Every call to `makeContext()` creates a distinct context. A `consume` only matches a `<Provide>` from the _same_ `makeContext()` call. There are no string keys, so there are no naming collisions: two addons can each have a context for "theme" and they cannot interfere with each other.

**Nearest provider wins.** `consume` walks up the render tree and returns the value from the closest enclosing `<Provide>` for its context:

```gjs
<template>
  <ctx.Provide @value="outer">
    {{ctx.consume}} {{! "outer" }}
    <ctx.Provide @value="inner">
      {{ctx.consume}} {{! "inner" }}
    </ctx.Provide>
  </ctx.Provide>
</template>
```

**Scoping follows the render tree.** A `<Provide>` is only visible to its descendants. Sibling subtrees do not see each other's providers, even for the same context. Separate `renderComponent` trees are fully isolated from each other. Because resolution follows the _render_ tree (not module structure, not the owner), a context provided in an app is visible inside a mounted engine's components, and a context can cross any component/helper boundary without anyone in the middle participating.

**Missing provider is an error.** `consume` throws if there is no enclosing `<Provide>` for its context, and throws if read outside of rendering entirely. This is intentional harm-reduction: a missing provider is almost always a bug, and `undefined` would let that bug travel. There is no "default value" feature -- if you want an always-available value, render a `<Provide>` at your application root. (Note that this is a place where this proposal is deliberately stricter than [ember-provide-consume-context][epcc], whose `getContext` returns `undefined`.)

**An explicit `undefined` is still provided.** `<ctx.Provide @value={{undefined}}>` (or omitting `@value`) means the provider _is_ in the tree and provides `undefined` -- `consume` returns `undefined` rather than throwing. "You provided nothing" and "there is no provider" are different situations, and only the latter is an error.

**Reactivity is just autotracking.** The `@value` binding is reactive: when the argument passed to `<Provide>` changes, consumers update. If `@value` is a stable object, mutating its `@tracked` fields updates consumers, and the object's identity is preserved across re-renders. There is no subscription API and no special invalidation mechanism -- a provided value behaves exactly like the same value passed as an argument.

**Where `consume` may be read.** Anywhere that runs synchronously during rendering, below a matching `<Provide>`:

- in a template, as a path: `{{ctx.consume}}`
- in a plain function helper's body
- in a component's constructor or getters

Because this is a synchronous, render-scoped API, `consume` does not work after an `await` (there is no render tree position to resolve against anymore -- it throws the "outside of rendering" error). Read the context first, then go async.

> [!NOTE]
> In the current implementation, `consume` inside a *modifier* also throws "outside of rendering", because modifiers run during the commit phase, after the render frame has unwound. 

### Testing

No test-support API is needed. Providing a context in a rendering test is the same as providing it anywhere else:

```gjs
test('FancyButton uses the provided theme', async function (assert) {
  const testTheme = new Theme();
  testTheme.color = 'hotpink';

  await render(
    <template>
      <theme.Provide @value={{testTheme}}>
        <FancyButton>hi</FancyButton>
      </theme.Provide>
    </template>
  );

  assert.dom('button').hasStyle({ color: 'rgb(255, 105, 180)' });
});
```

### How it works (non-normative)

The renderer already tracks the render-tree hierarchy (this is how the debug render tree behind Ember Inspector works). The implementation maintains a similar, always-on tracker of "scope nodes" as components are created and re-rendered. `<Provide>` records its (lazily-read, autotracked) `@value` on the current node under its context's private identity; `consume` walks the current node's parent chain looking for that identity.

None of this is public API. The tracker, its module, and its shape may change freely as the renderer evolves -- the only stable surface is `makeContext` and the behavior specified above. (This is the most important difference from [RFC #1154][rfc-1154], which would have made the tree-walking itself public.)

### Ecosystem implications

- a rule in `eslint-plugin-ember` flagging `consume` (and other render-scoped reads) after an `await` would be a nice addition, but is not required for this feature to ship.
- [ember-provide-consume-context][epcc] can migrate its internals onto this (or document a migration path for its users) and stop overriding VM internals.

### What this RFC does _not_ propose

None of the following is proposed here:

- no `getScope` / `addToScope` or any other render-tree exploration API (that was [RFC #1154][rfc-1154]; this supersedes it)
- no `@provide` / `@consume` decorators and no string-keyed contexts (sketched in [RFC #975][rfc-975])
- no changes to component manager APIs (the approach in [RFC #1155][rfc-1155])
- no changes to `getOwner` or any owner-related API (see the appendix for why this is worth mentioning)
- no default values for contexts
- no test-support module

If any of these turn out to be wanted later, they can be their own (small) RFCs on top of this one.

## How we teach this

The terminology is "context", "provider", and "consumer" -- the same words React, Vue (`provide`/`inject`), and Svelte (`setContext`/`getContext`) users already have for this concept. This is a case where matching the rest of the ecosystem is a feature: many developers will arrive already knowing what a context is, and the ones who don't will find a decade of general material about the pattern.

In the guides, context belongs after services, as "like a service, but scoped to part of the render tree instead of the whole application, with its lifetime tied to the provider's place in the tree." That comparison does most of the teaching:

| | service | context |
|---|---|---|
| visibility | whole application | descendants of a `<Provide>` |
| key | string (module name) | object identity (`makeContext()` result) |
| lifetime | application | the provider's block |
| can have many simultaneous values | no | yes (one per `<Provide>`) |

Guidance on _when_ to reach for it matters as much as the mechanics: app-wide state should remain a service; passing data one or two levels down should remain plain arguments; context is for when a component family or a region of the app needs shared ambient state and threading arguments through every intermediate component is the thing being avoided.

The API docs come from the JSDoc on `makeContext` (already written in the implementation PR) and should lead with a complete provide-then-consume example, plus the two error cases, since the throwing behavior is the part most likely to surprise someone coming from [ember-provide-consume-context][epcc].

## Drawbacks

- It is one more state-sharing tool to choose between (arguments, yields/contextual components, services, and now context), and it can be misused -- overly-broad context values cause the same problems in every framework that has the feature. This is a teaching problem, and the guides section above is most of the mitigation.
- `consume` throwing on a missing provider is stricter than what existing [ember-provide-consume-context][epcc] users are used to, so migrating code may surface latent "consumed but never provided" bugs. (We think surfacing those is the point, but it is a real migration bump.)
- The always-on scope tracker adds a small amount of bookkeeping to every component create/update, whether or not an app uses context. The implementation keeps this to a stack push/pop with lazy allocation of everything else, and the implementation PR's smoke tests exist to keep it honest.

## Alternatives

**Do nothing.** The community keeps depending on a VM-internals override that ember-source has to tiptoe around. 

**[RFC #975][rfc-975]: `@provide` / `@consume` decorators with string keys.** This was the original proposal and it shaped this one. Its API has three problems this design avoids: string keys collide across addons; decorators only serve class components (template-only components, function helpers, and strict-mode templates are left out); and it coupled the design to the services API ("if services change form, context changes with it"). `makeContext` works in every component form, keys by identity, and stands alone. The decorators could still be built _on top of_ this primitive by anyone who wants them.

**[RFC #1154][rfc-1154]: public `getScope` / `addToScope`.** The previous attempt, by this RFC's author. It proposed the general capability (walk the render tree's userland metadata) so that Context could be explored in userland. Implementation showed that the general capability is the expensive thing to stabilize -- iteration order, entry shapes, owner access, and reactivity caveats all become public commitments -- while the thing people want to build with it took an API one-tenth the size. This RFC keeps the same underlying machinery private and ships the feature instead.

**[RFC #1155][rfc-1155]: expose the render tree to component managers.** This only serves components (helpers and other invokables can't participate), requires addon authors to interact with manager APIs to use it, and exposes internal manager values that aren't designed for extension.

**`consume` as a function instead of a getter.** Earlier drafts (and the implementation PR) made `consume` a function. A function signals "this runs work and can throw" more loudly than a property access. But the getter composes with template paths (`{{theme.consume.color}}`) and reads as what it is -- a value you read, autotracked like any other -- and the throwing and render-scoped restrictions are identical either way. Nothing about the underlying machinery depends on the choice; this is purely the public spelling.

**Prior art.** [ember-provide-consume-context][epcc] (whose test suite the implementation PR ports, so its production-proven behaviors -- sibling isolation, conditional providers, reactivity to value changes -- are pinned down as this feature's behavior), [React Context](https://react.dev/learn/passing-data-deeply-with-context), [Vue provide/inject](https://vuejs.org/guide/components/provide-inject.html), [Svelte setContext/getContext](https://svelte.dev/docs/svelte#setcontext), and [ember-context](https://github.com/alexlafroscia/ember-context) (which doesn't follow the render tree, and documents the resulting caveats).

## Unresolved questions

- **Module home.** The implementation exports from `@ember/helper`, since `consume` is helper-shaped and that module already exports the template utilities (`fn`, `hash`, `array`, ...). A dedicated `@ember/context` module is the plausible alternative. The behavior in this RFC is unaffected either way.
- **Naming.** `makeContext` vs `createContext` (React's name), and `Provide` vs `Provider`. The implementation deliberately does not reuse React's exact names, since the shapes differ (`createContext(defaultValue)` vs `makeContext()` with no default), but bikeshedding is welcome.

## Appendix: what this primitive makes cheap (not proposed here)

> [!IMPORTANT]
> Nothing in this appendix is proposed by this RFC. It records where related exploration lives so that the proposal above can stay minimal.

A long-wanted capability is `getOwner()` with **no arguments** working inside plain function helpers, which have no `this` to read an owner from:

```gjs
import { getOwner } from '@ember/owner';

function currentLocale() {
  return getOwner()?.lookup('service:intl').locale;
}

<template>
  {{ (currentLocale) }}
</template>
```

It turns out the owner _is_ a context: a value that everything below a point in the render tree should be able to read, where the nearest provider wins (which is exactly what `renderComponent`'s `owner` option and engine mounts need). Built on this RFC's machinery, the whole feature is essentially three lines -- a well-known internal context key, the helper opcodes providing the active owner under it, and `getOwner()` reading it back:

```ts
// a well-known (internal) context key
export const OWNER: object = {};

// at helper invocation, in the renderer
provideRenderContext(OWNER, () => owner);

// in getOwner(), when called with no argument
let read = lookupRenderContext(OWNER);
return read ? read() : undefined;
```

A working implementation of this, stacked on the implementation of this RFC, is at [NullVoxPopuli/ember.js#16](https://github.com/NullVoxPopuli/ember.js/pull/16) -- and because the owner lookup walks the render tree like any other context, it resolves correctly through `renderComponent` owner overrides and re-renders, for free.

If/when no-arg `getOwner` is wanted, it will be its own RFC; it would change the public signature of `getOwner`, and deserves its own discussion.
