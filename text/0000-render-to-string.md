---
stage: accepted
start-date: 2026-07-02T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite:
---

# `renderToString`

## Summary

Add `renderToString` to `@ember/renderer`: the server-side-rendering counterpart to [`renderComponent`](https://github.com/emberjs/rfcs/blob/main/text/1099-renderComponent.md). It renders a component through the same pipeline as `renderComponent` and resolves to the serialized HTML once rendering has settled. A companion `env.rehydrate` option on `renderComponent` completes the round trip on the client.

## Motivation

[RFC #1099](https://github.com/emberjs/rfcs/blob/main/text/1099-renderComponent.md) gave us a public API for rendering a component into any element, enabling integration with Astro, Vitepress, Storybook, and buildless demos. All of those environments also have a server story, and Ember has no public primitive for it.

Today the only way to get HTML out of Ember on a server is FastBoot, which:

- renders whole apps, not components — it cannot serve the "islands" integrations that `renderComponent` enables
- is built on SimpleDOM, forcing a degraded render: `isInteractive: false`, no modifiers, no settling, and a pile of "the server is different" special cases that leak into app code

`renderToString` takes the opposite position: **a server render is a real render, not a degraded one**. The same renderer machinery, real DOM, modifiers running, reactivity settling — then `innerHTML`.

This gives meta-frameworks (Astro/Vitepress server entrypoints, Vite SSR) and app authors a small, composable SSR primitive, and — together with `env.rehydrate` on `renderComponent` — a full server-render → client-adopt round trip.

## Detailed design

Users would import `renderToString` from `@ember/renderer` (a pre-existing module).

The interface:

```ts
/**
 * Renders a component and resolves to its serialized HTML
 * once rendering has settled.
 */
export async function renderToString(
    /**
     * The component definition to render.
     *
     * Any component that has had its manager registered is valid,
     * same as `renderComponent`.
     */
    component: object,
    options?: {
        /**
         * Optional owner. Defaults to `{}`, can be any object, but will need to
         * implement the [Owner](https://api.emberjs.com/ember/release/classes/Owner)
         * API for components within this render tree to access services.
         */
        owner?: object;

        /**
         * These args get passed to the rendered component.
         *
         * If your args are reactive, rendering settles before serialization —
         * updates made during render are reflected in the output.
         */
        args?: Record<string, unknown>;

        /**
         * Optionally configure the rendering environment
         */
        env?: {
            /**
             * When true, the emitted HTML includes glimmer's rehydration
             * markers so a subsequent client-side render can adopt the markup
             * instead of rebuilding it. Defaults to false, which produces
             * clean HTML with no framework-specific comments.
             */
            rehydratable?: boolean;
        };
  }
): Promise<string> {
    /* ... implementation details ... */
}
```

When called, `renderToString`:

1. renders the component into a detached element, through the same renderer machinery as `renderComponent` (error-loop protection, transactions, destroyables)
2. awaits `renderSettled()` — if a modifier (or anything else during render) updates tracked state, the re-render completes before serialization
3. serializes the result via `innerHTML`
4. tears the render tree down (running destructors) before the promise resolves

### A server render is a real render

`renderToString` deliberately has no `isInteractive`, `hasDOM`, or `document` options — a server render behaves exactly like a browser render:

- **Modifiers run**, against real elements, and their DOM effects (attributes, etc.) are serialized into the output.
- **The environment provides the document**, same as in the browser. There is no SimpleDOM and no `document` option. In Node, register real DOM building blocks globally first — for example with [happy-dom](https://github.com/capricorn86/happy-dom):

```js
import { GlobalRegistrator } from '@happy-dom/global-registrator';
import { renderToString } from '@ember/renderer';

GlobalRegistrator.register(); // once, in Node — the browser already has a document

let html = await renderToString(MyComponent, { args: { name: 'Zoey' } });
// => "<h1>Hello, Zoey!</h1>"
```

Calling `renderToString` without a global `document` is an error, with a message pointing at the above.

This keeps the server code path identical to the browser code path (`instanceof` checks and all), and means nothing new ships in the published build — serialization is just `innerHTML`.

Requiring a real DOM is not a limitation to work around — it corrects a fundamental design mistake in Ember's existing SSR support. The `isInteractive` and `hasDOM` flags exist only because SimpleDOM cannot behave like a real document; each one is a fork between the server and browser code paths that app and addon code then has to guard against. With a real DOM there is nothing to flag. Modifiers running during a server render are most often a no-op, and at best beneficial — setting styles, manipulating state — with their DOM effects serialized like any other output.

### The Parameters

#### `owner` (defaults to `{}`)

Same as `renderComponent`.

#### `args`

The args to pass to the component. Rendering settles before serialization, so tracked updates made during render are reflected in the returned HTML.

#### `env.rehydratable` (defaults to `false`)

When true, the output includes glimmer's rehydration markers, for consumption by a client-side rehydrating render (below). When false, the output is clean HTML.

### Rehydration: `env.rehydrate` on `renderComponent`

This RFC also adds one option to `renderComponent`'s `env`:

```ts
env?: {
    isInteractive?: boolean;

    /**
     * When true, the render adopts (rehydrates) server-rendered markup
     * already present in `into` — produced by `renderToString` with
     * `env: { rehydratable: true }` — instead of clearing it and building
     * fresh DOM.
     */
    rehydrate?: boolean;
};
```

The round trip:

```js
// server
let html = await renderToString(MyComponent, { args, env: { rehydratable: true } });

// client — adopts the server-rendered DOM instead of rebuilding it
renderComponent(MyComponent, { into: element, args, env: { rehydrate: true } });
```

A rehydrating render adopts the existing DOM nodes — same node identity — rather than replacing them. Internally, the tree builder is chosen per render call (not per renderer), so a prior normal render never locks an owner out of rehydrating later. `rehydratable` is `renderToString`-only, just as `rehydrate` is `renderComponent`-only.

### FastBoot

FastBoot and `renderToString` do not interoperate: FastBoot's SimpleDOM-based model is exactly what this design rejects. FastBoot would need to be updated to build on `renderToString` — dropping SimpleDOM for a real DOM implementation and adopting settle-before-serialize. That work would land in FastBoot, not ember-source, and is out of scope for this RFC.

### Usage

An HTTP handler in Node:

```js
import { GlobalRegistrator } from '@happy-dom/global-registrator';
import { renderToString } from '@ember/renderer';
import { createServer } from 'node:http';

import { ProductPage } from './components/product-page.js';

GlobalRegistrator.register();

createServer(async (req, res) => {
    let html = await renderToString(ProductPage, {
        args: { id: new URL(req.url, 'http://x').searchParams.get('id') },
        env: { rehydratable: true },
    });

    res.end(`<!doctype html><div id="app">${html}</div><script type="module" src="/client.js"></script>`);
}).listen(3000);
```

```js
// client.js — adopt the server-rendered DOM
import { renderComponent } from '@ember/renderer';
import { ProductPage } from './components/product-page.js';

renderComponent(ProductPage, {
    into: document.querySelector('#app'),
    args: { id: new URL(location.href).searchParams.get('id') },
    env: { rehydrate: true },
});
```

An Astro server entrypoint (the counterpart to RFC #1099's client entrypoint):

```ts
// astro/packages/integrations/ember/src/server.ts
import { renderToString } from '@ember/renderer';

export default {
    async renderToStaticMarkup(Component: object, props: Record<string, unknown>) {
        return { html: await renderToString(Component, { args: props }) };
    },
};
```

## How we teach this

`renderToString` is taught as the server half of `renderComponent` — same arguments, minus `into`, returning a promise of a string. Anyone who knows `renderComponent` already knows `renderToString`.

API docs live in `@ember/renderer` alongside `renderComponent`. A guides page on server-side rendering would cover:

- registering a global DOM implementation in Node (happy-dom)
- the render → settle → serialize lifecycle (and that destructors run before the promise resolves)
- the rehydration round trip: `rehydratable` on the server, `rehydrate` on the client

The one genuinely new concept is "the environment provides the document." That is one line of setup in Node, and no setup in the browser — much less to teach than FastBoot's catalog of server/browser differences (`isFastBoot` guards, no-modifiers, SimpleDOM limitations).

## Drawbacks

- Serialization is all-at-once. A streaming API is desired, but would need to be a separate RFC.
- Until FastBoot is updated, Ember has two SSR stories that don't interoperate.

## Alternatives

There is an existing implementation in [PR #21481](https://github.com/emberjs/ember.js/pull/21481).

An alternative implementation exists in [PR #21480](https://github.com/emberjs/ember.js/pull/21480), which bypasses the renderer machinery and drives glimmer's low-level `renderComponent` directly. This RFC proposes the #21481 shape because building on the renderer pipeline shares error-loop protection, transactions, and destroyables with `renderComponent` — one pipeline to maintain, and behavioral parity between client and server renders by construction.

Other designs considered:

- **A `document` option (SimpleDOM-style), as FastBoot does.** Rejected: it reintroduces the "server is different" split, requires shipping serialization code in the build, and breaks `instanceof` checks. The global-document requirement keeps one code path.
- **Extending FastBoot instead.** FastBoot is whole-app; the motivating use cases (islands, meta-framework entrypoints) are per-component. Component-level SSR is the primitive; whole-app SSR can be built on top of it, not the other way around.
- **Doing nothing.** `renderComponent` integrations remain client-only, and every framework Ember competes with ships a `renderToString` equivalent (React's `renderToString`, Vue's `@vue/server-renderer`, Svelte's `render`).

## Unresolved questions

- Should settling integrate with test-waiter-style waiters (beyond `renderSettled()`), so data-loading patterns can hold serialization open until data resolves?
- Is `rehydrate` the right knob on `renderComponent`, or should rehydration be a separate entry point?
