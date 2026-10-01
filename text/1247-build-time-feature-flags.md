---
stage: accepted
start-date: 2026-09-30T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - framework
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1247
project-link:
suite:
---


# Build-time Feature Flags

## Summary

`ember-source` reads every flag from `import.meta.env?.EMBER_*`, and from nowhere else:

```js
// @ember/-internals/environment
export const DEFAULT_ASYNC_OBSERVERS = import.meta.env?.EMBER_DEFAULT_ASYNC_OBSERVERS ?? true;
```

The bundler replaces the expression with a literal.
Every read of `DEFAULT_ASYNC_OBSERVERS` then folds to a constant, and the minifier removes the code that the app does not use.
This covers canary features, optional features, `EmberENV`, and deprecations that can remove their code early.

Existing config files keep working, because the build reads them and turns them into `import.meta.env` values.
Minimal apps with no config files get a way to set flags for the first time.
The runtime global `window.EmberENV` stops being a source of flags.

## Motivation

Today, all of Ember's flags are read at runtime from `window.EmberENV`, so every branch ships to every user:
- an app that turned on `default-async-observers` years ago still ships the sync observer path
- an app that turns off `EmberObject` ([RFC 1234](./1234-deprecate-ember-object.md)) would still ship `EmberObject`

A build-time value with a runtime fallback (`import.meta.env?.X ?? EmberENV.X`) does not fix this, because any flag that the build does not set keeps both branches.
Dead-code removal requires one source of truth.

Minimal apps, such as [`v2-app-hello-world-template`](https://github.com/emberjs/ember.js/tree/main/smoke-tests/v2-app-hello-world-template) and the [`ember.nvp`](https://github.com/NullVoxPopuli/ember.nvp) `minimal-app`, have no `config/environment.js` and no `@embroider/virtual/vendor.js`, so they have no way to set a flag at all.

## Detailed design

Each flag is an exported `const`, and `ENV` is built from those:

```js
// @ember/-internals/environment
export const DEFAULT_ASYNC_OBSERVERS = import.meta.env?.EMBER_DEFAULT_ASYNC_OBSERVERS ?? true;
export const DEBUG_RENDER_TREE = import.meta.env?.EMBER_DEBUG_RENDER_TREE ?? DEBUG;
export const FEATURE_FOO_BAR = import.meta.env?.EMBER_FEATURE_FOO_BAR ?? false;

export const ENV = {
  _DEFAULT_ASYNC_OBSERVERS: DEFAULT_ASYNC_OBSERVERS,
  _DEBUG_RENDER_TREE: DEBUG_RENDER_TREE,
  FEATURES: { FOO_BAR: FEATURE_FOO_BAR },
};
```

- The `?.` is for environments without `import.meta.env` (plain `<script type="module">`, import maps, Node). Those get the default.
- The default sits next to each flag, which decides if the flag is opt-in or opt-out.
- Internal code imports the `const`, not `ENV.X`. Vite 8 (Rolldown) does not fold property reads, even on an object that nothing writes to. Both Vite 7 and Vite 8 fold a `const`.
- `ENV` stays (for the Inspector and `isEnabled`), but nothing writes to it, and the `for (flag in EmberENV)` merge goes away.

The name is `EMBER_` plus the existing name in `SCREAMING_SNAKE_CASE`, without the leading underscore.
Canary features get `FEATURE_` because their names are free-form.
Values are JSON (`EMBER_RERENDER_LOOP_LIMIT` is a number, for example).

### Setting flags

```bash
# .env
EMBER_DEBUG_RENDER_TREE=false
```

```js
// vite.config.mjs
export default defineConfig({
  define: {
    'import.meta.env.EMBER_DEBUG_RENDER_TREE': 'false',
  },
});
```

Either one removes the whole debug render tree, in development too.
Today it always ships, because development builds force it on.

The `ember()` plugin from `@embroider/vite`:
- reads `config/environment.js` (`EmberENV`) and `config/optional-features.json`, and defines the matching `EMBER_*` values. Existing apps get smaller builds with no changes.
- parses `.env` values as JSON. Plain Vite gives the string `"false"`, which is truthy, so without the plugin, use `define`.
- resolves conflicts as `define`, then `.env`, then `config/environment.js`.

Minimal apps have no config files, so they use `.env` or `define` like any other Vite config.
Generators can drop the `EmberENV` key from `app/config.ts` (in `ember.nvp`, nothing reads it).

### `window.EmberENV`

`ember-source` no longer reads `window.EmberENV`.
Most apps won't notice, because Embroider fills it from the same `config/environment.js` that the plugin reads.
Apps that set it by hand lose those values.
For those, development builds throw an error for each mismatched key, naming the `EMBER_*` flag to use.

The same goes for runtime writes to `FEATURES` from `@ember/canary-features`.
`isEnabled` stays public, but internal code uses the `FEATURE_*` constants, because `isEnabled('FOO')` can't fold.
Beta and release builds of `ember-source` replace `EMBER_FEATURE_*` with the channel's value, so unfinished features stay out of the tarball (same as today).

### Optional features

`default-async-observers` is the only optional feature that `ember-source` still reads.
Its default becomes `true`, matching the app blueprint.
Apps that don't list it in `optional-features.json` have sync observers today, so for them the plugin defines `false`.

> [!NOTE]
> Observers are deprecated ([RFC #1115](https://github.com/emberjs/rfcs/pull/1115)) and removed in v8, along with `EMBER_DEFAULT_ASYNC_OBSERVERS`.

`@ember/optional-features` is not deprecated by this RFC.

### Sveltable deprecations

How deprecations work does not change.

Some deprecations come with a flag that removes the deprecated feature before the major that removes it (for example, `deprecate-ember-object` from [RFC 1234](./1234-deprecate-ember-object.md)).
These get a `const` named after the deprecation `id`:

```js
export const DEPRECATION_DEPRECATE_EMBER_OBJECT =
  import.meta.env?.EMBER_DEPRECATION_DEPRECATE_EMBER_OBJECT ?? false;
```

`true` removes the code.
The default stays `false` until the major where that deprecation's RFC turns it on.

### Ecosystem

- `ember-auto-import` must define the `EMBER_*` values from the config files, or classic apps lose their config. This is required before `ember-source` ships the change.
- v2 addons can use the same pattern for their own flags.
- TypeScript: `ember-source` declares the `EMBER_*` keys on `ImportMetaEnv`.
- SSR, FastBoot, and the Inspector: no change.

## How we teach this

The [feature flags](https://guides.emberjs.com/release/configuring-ember/feature-flags/) and optional features guides show `.env` and `define` first, and `config/environment.js` as the classic option.

The deprecation guide for each sveltable deprecation documents which environment variable removes the deprecated code.

## Drawbacks

- Apps that set `window.EmberENV` outside of `config/environment.js` must move that config into the build. The error in development builds finds each case.
- A flag can't change at runtime, so a test suite that toggles a flag needs one build per value, like the ember.js CI does for `ALL_DEPRECATIONS_ENABLED` today. (With `define`, you can bring back runtime behavior, however.)
- Every build that wants non-defaults must define the `EMBER_*` values.
- Without a bundler, there is no way to set a flag.

## Alternatives

- `@embroider/macros` (`getOwnConfig`, `macroCondition`). Works today, but it is Ember-specific and requires Babel. `import.meta.env` is the ecosystem standard.
- Do nothing. Apps keep paying for code paths that they turned off.

## Unresolved questions

n/a
