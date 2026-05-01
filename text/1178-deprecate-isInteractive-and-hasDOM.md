---
stage: accepted
start-date: 2026-05-01T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1178
project-link:
---

# Deprecate `isInteractive` and `hasDOM` from the renderer

## Summary

Deprecate the `isInteractive` and `isBrowser` boot options on `ApplicationInstance`, the `hasDOM` and `isInteractive` flags exposed on the renderer environment, and the rendering codepaths that branch on them. After this deprecation resolves, `ember-source` will only support running its renderer in a browser-like environment (a real browser or a JSDOM-style environment that provides a `Document` and DOM events).

## Motivation

These flags exist to support a non-interactive, non-browser rendering mode that was originally added for FastBoot-style server-side rendering. FastBoot was only a "best effort" sort of support, never officially documented in the guides, or recommended as a good SSR solution (it did happen to be the _only_ SSR solution for a while). Fastboot has not kept up with current Ember and is incompatible with the Vite-based build pipeline, and the codepaths it required impose cost on every consumer:

- `Renderer`, the curly component manager, the `Input` built-in, the event dispatcher, and `ApplicationInstance` all carry branches that exist only to handle `isInteractive === false` or `hasDOM === false`.
- The `BootEnvironment` shape spreads `@ember/-internals/browser-environment` (which itself only exists to expose `hasDOM`) into the renderer for compatibility.
- Component lifecycle hooks (`didInsertElement`, `didRender`, etc.) are silently skipped in non-interactive mode, which is a surprising semantic for any app that accidentally lands there.

There is now a _broader-ecosystem_ path for SSR with current Ember that does not require these flags: [`vite-ember-ssr`](https://github.com/evoactivity/vite-ember-ssr) renders in normal interactive mode and can even enable the use of `settled()` to await rendering. Removing the non-interactive mode lets us delete a substantial amount of renderer cruft (see [emberjs/ember.js#21348](https://github.com/emberjs/ember.js/pull/21348), which removes ~1700 lines) while leaving SSR consumers with a working, modern replacement.

The replacement functionality is: render in interactive mode against a DOM (real or JSDOM) and, for SSR, `await settled()` before serializing the result, as demonstrated by `vite-ember-ssr`.

> [!IMPORTANT]
> This RFC does not explicitly deprecate fastboot, but it deprecates the technique that fastboot currently relies on for SSR/SSG and it will need to move to a more modern approach if it wants to remain in use.

## Transition Path

The following public API surface is deprecated. Each item should issue a deprecation when used, and continue to behave as it does today until the deprecation is removed in a subsequent major.

### `BootOptions.isInteractive`

Passing `isInteractive` to `ApplicationInstance#boot`, `ApplicationInstance#visit`, or `Application#visit` is deprecated. The option was always documented as `@private` but was reachable through the public `visit` API.

Before:

```js
application.visit('/', { isInteractive: false });
```

After: omit the option. If you need to render without attaching event listeners, render into a detached document (e.g. JSDOM) instead of a live page; the renderer will produce the same HTML.

### `BootOptions.isBrowser`

Passing `isBrowser` to `ApplicationInstance#boot`, `ApplicationInstance#visit`, or `Application#visit` is deprecated. `isBrowser: false` was the documented switch for "non-browser" rendering and forced `isInteractive` to `false` and `location` to `'none'`.

Before:

```js
application.visit('/', { isBrowser: false, document: simpleDom });
```

After: provide a DOM via the `document` option (a real `Document` or a JSDOM `Document`) and, if desired, set `location: 'none'` explicitly. There is no longer a "non-browser" mode; the renderer always assumes a DOM-like host.

### Non-`Document` values for `BootOptions.document`

`BootOptions.document` previously accepted `Document`-like objects with only a partial DOM surface (the `SimpleDOM` library was called out as known-good). It now requires a value that implements the DOM APIs the renderer uses in interactive mode (`addEventListener`, `createElement`, etc.). Passing a non-`Document` value is deprecated.

After: use a real `Document`, or a JSDOM-style `Document` polyfill, for non-browser hosts.

### `hasDOM` and `isInteractive` on the renderer environment

The `hasDOM` and `isInteractive` properties on the `BootEnvironment` / `Environment` object passed through the renderer are deprecated. Reading them is deprecated; both will be removed and the renderer will behave as if `hasDOM === true` and `isInteractive === true`.

This also deprecates the `@ember/-internals/browser-environment` entry point, which exists solely to publish `hasDOM`. It is an internal package and not part of the public API, but is called out here because addons have historically reached into `@ember/-internals/*`.

### Component-lifecycle behaviour

Component lifecycle hooks (`didInsertElement`, `didRender`, `willRender`, `didReceiveAttrs`, etc. on classic components) currently no-op when `isInteractive === false`. Once `isInteractive` is removed, these hooks will always run. Apps that today rely on these hooks *not* running in their FastBoot-style boot path should either not invoke that boot path or guard the hook bodies themselves.

### Deprecation guide

A deprecation guide should be published at `https://deprecations.emberjs.com/` covering:

- Pointing FastBoot users at `vite-ember-ssr` as the supported SSR path.
- In order to do this migration, users must:
  - disable FastBoot
  - complete their vite migration
  - follow the README for vite-ember-ssr (for the time being)

> [!NOTE]
> vite-ember-ssr is not currently "formally supported" by the core team, but is the SSR solution based on vite, which is not built on so many hacks as fastboot was.

> [!IMPORTANT]
> Details of this migration have been done for ember-page-title, tho will vary depending on your project, addon, etc, and the list of PRs that made up the migration can be [found here](https://github.com/ember-cli/ember-page-title/pull/308)

### Ecosystem implications

- **FastBoot**: FastBoot was never formally supported, but is the primary real-world consumer of these flags. Its maintainers, and any apps still on FastBoot, should plan a migration to vite-based-SSR, such as `vite-ember-ssr` (or accept that they cannot upgrade past the major in which these flags are removed).
- **Addons**: addons would have primarily only incidentally encountered these deprecated features via fastboot 
- **Ember Inspector**: no impact.
- **Ember Engines**: `EngineInstance#boot` accepts the same `BootOptions`; the deprecation applies there as well.
- **Blueprints**: no changes needed; blueprints do not generate code that uses these options.
- **IDE / TypeScript**: the public type for `BootOptions` should mark `isInteractive` and `isBrowser` as `@deprecated` so editors surface the warning at call sites.

For console or other rendering environments, they should be unaffected, because they are also implementing a browser of sorts.
If/when we want to formally support different renderers (rather than "anything that can be browser like"), we can design an API that allows the whole renderer to be swapped out, and not bundled).

## How We Teach This

The Ember Guides do not currently teach `isInteractive` or `isBrowser`; they are mentioned only in API docs for `Application#visit` and `BootOptions`. The teaching work is therefore concentrated in the API docs and the deprecation guide:

- Update the `Application#visit` and `BootOptions` API docs to mark `isInteractive` and `isBrowser` as deprecated, and to remove the "non-browser mode" prose around `document` and `rootElement`.
- Publish the deprecation guide described above.
- In any SSR-adjacent material, point readers at Vite-based-SSR, such as `vite-ember-ssr` rather than implying that `ember-source` itself ships a non-browser renderer mode.

For existing users, the framing is: "Ember's renderer assumes a DOM. If you need SSR, render against a DOM (JSDOM, or a real browser) using a tool like `vite-ember-ssr`; the standalone `isInteractive` / non-browser switch is going away."


## Drawbacks

- Apps still running on FastBoot cannot upgrade past the major that removes these flags without first migrating to `vite-ember-ssr` or dropping SSR. There is real migration cost for those users, even though FastBoot was never officially supported.
- Any addon that intentionally skipped work during FastBoot rendering will now run that work during SSR. Most such addons should be fine — `vite-ember-ssr` runs in interactive mode, which is exactly what those addons used to opt out of — but a few may need updates.
- We are formalising "ember-source only supports browser-like environments" as a constraint. Future SSR work that wants to render without a DOM at all (e.g., a streaming string renderer) would have to reintroduce a similar split.

## Alternatives

- **Do nothing.** Keep `isInteractive`/`hasDOM` indefinitely. This preserves theoretical FastBoot compatibility but leaves the renderer carrying branches for a path that is not formally supported, not tested against current Ember, and not compatible with the Vite build that the ecosystem is moving to.
- **Remove without deprecation.** These are semi-private APIs, and fastboot's current CI is all failing, so it's not clear that fastboot even works with current ember.

## Unresolved questions

n/a
