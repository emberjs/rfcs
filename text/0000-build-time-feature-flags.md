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
  accepted: # Fill this in with the URL for the Proposal RFC PR
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

Ember has three places to configure the framework, and all three are runtime-only:

| Source | Where the app sets it | How `ember-source` reads it |
| --- | --- | --- |
| Canary features | `EmberENV.FEATURES` in `config/environment.js` | `isEnabled('FLAG')` from `@ember/canary-features` |
| Optional features | `config/optional-features.json` | `@ember/optional-features` copies the values into `EmberENV` |
| `EmberENV` | `config/environment.js` | `ENV` from `@ember/-internals/environment` |

All three end up in `window.EmberENV`.
In an app with `@embroider/compat`, Embroider writes the `EmberENV` object from `config/environment.js` into `/@embroider/virtual/vendor.js`, which runs before Ember loads.
Because the values arrive at runtime, every branch ships to every user.
An app that turned on `default-async-observers` years ago still downloads the sync observer path.
[RFC 1234](./1234-deprecate-ember-object.md) adds a flag that turns off `EmberObject`, but the code still ships until the build can remove it.

A build-time value alone does not fix this.
Assume that `ember-source` reads a flag from `import.meta.env` and falls back to `window.EmberENV`.
Then each flag that the build does not set still depends on a runtime value, and both branches stay.
One source of truth is a requirement for dead-code removal.

Vite, Rolldown, esbuild, and Rollup all replace `import.meta.env.*` with literals at build time.
If Ember reads its flags from `import.meta.env`, then the tools that apps already use can remove dead code, with no Ember-specific plugin in the minifier.

Minimal apps have no path at all.
The [`v2-app-hello-world-template`](https://github.com/emberjs/ember.js/tree/main/smoke-tests/v2-app-hello-world-template) in the ember.js repo and the `minimal-app` output of [`ember.nvp`](https://github.com/NullVoxPopuli/ember.nvp) have no `@embroider/compat`, no `config/environment.js`, and no `vendor.js`.
The `ember.nvp` output has an `EmberENV: {}` key in `app/config.ts`, but nothing copies it to `window.EmberENV`, so it has no effect.
The only way for these apps to set a flag today is a hand-written `window.EmberENV` assignment that runs before the first import of `ember-source`.

`ember-source` already uses this pattern internally.
`@glimmer/local-debug-flags` reads `import.meta.env?.VM_LOCAL_DEV`.

## Detailed design

### The read pattern

Each flag is one exported `const`, and `ENV` is built from those constants:

```js
// @ember/-internals/environment
export const FLAG = import.meta.env?.EMBER_FLAG ?? <default>;
export const FEATURE_FOO_BAR = import.meta.env?.EMBER_FEATURE_FOO_BAR ?? false;

export const ENV = {
  FLAG,
  FEATURES: { FOO_BAR: FEATURE_FOO_BAR },
};
```

Code in `ember-source` imports the `const`, and does not read `ENV.FLAG`.
`ENV` stays for Ember Inspector and for `isEnabled`, and it holds the same values.

- `import.meta.env?.` uses optional chaining because some environments do not define `import.meta.env`. Examples: a plain `<script type="module">`, an import map, or Node without a bundler. In those environments the expression is `undefined`, and the default applies.
- The default lives next to the flag in `ember-source`. This is how Ember decides if a flag is opt-in or opt-out, per flag.

A property read does not fold reliably.
Vite 7 (Rollup) folds `ENV.X` only while nothing writes to `ENV` and `ENV` does not escape through `getENV()`.
Vite 8 (Rolldown) does not fold `ENV.X` at all, even when the object is never written to.
Both versions fold a `const` export.

Two more changes follow from one source of truth:

1. The `for (flag in EmberENV)` loop that copies `window.EmberENV` into `ENV` goes away.
2. Nothing writes to `ENV`. For example, `@ember/application` sets `ENV.LOG_VERSION = false` so that the version logs once. That state moves to a module-local variable.

Values are JSON: booleans for most flags, and a number or string for the few that need it (`EMBER_RERENDER_LOOP_LIMIT`, `EMBER_OVERRIDE_DEPRECATION_VERSION`).

### Names

The name is `EMBER_` plus the existing name in `SCREAMING_SNAKE_CASE`, with the leading underscore removed.

| Today | Build-time name |
| --- | --- |
| `EmberENV.FEATURES.FOO_BAR` | `EMBER_FEATURE_FOO_BAR` |
| `default-async-observers` in `optional-features.json` | `EMBER_DEFAULT_ASYNC_OBSERVERS` |
| `EmberENV.LOG_VERSION` | `EMBER_LOG_VERSION` |
| `EmberENV._DEBUG_RENDER_TREE` | `EMBER_DEBUG_RENDER_TREE` |
| `EmberENV._ALL_DEPRECATIONS_ENABLED` | `EMBER_ALL_DEPRECATIONS_ENABLED` |

Canary features get a `FEATURE_` segment, because their names are free-form and can collide with `EmberENV` keys.

### How an app sets a flag

Existing config keeps working, and it also becomes build-time config.
The `ember()` Vite plugin from `@embroider/vite` reads `config/optional-features.json` and the `EmberENV` object from `config/environment.js`, and it defines the matching `import.meta.env.EMBER_*` values.
So an existing app gets smaller output after it upgrades, with zero changes.
These files are inputs to the build, and `ember-source` never reads them at runtime.

An app can also set flags with the tools that Vite users already know:

```bash
# .env
EMBER_DEBUG_RENDER_TREE=false
```

```js
// vite.config.mjs
export default defineConfig({
  define: {
    'import.meta.env.EMBER_LOG_INSPECTOR_HINT': 'false',
  },
});
```

`EMBER_DEBUG_RENDER_TREE=false` removes the debug render tree from the build, in development too.
Today `_DEBUG_RENDER_TREE` defaults to `DEBUG`, and development builds force it to `true`, so the code always ships.
An app that does not use Ember Inspector or `captureRenderTree` does not need it.

The `ember()` plugin reads `EMBER_*` keys from `.env` files and defines each one as a JSON value.
Plain Vite gives `.env` values to the app as strings, so `EMBER_DEBUG_RENDER_TREE=false` is the string `"false"`, and that string is truthy.
Without the `ember()` plugin, use `define`.
If `config/environment.js` and `.env` or `define` set the same flag, `define` wins, then `.env`, then `config/environment.js`.

#### Minimal apps

A minimal app has no `config/environment.js` and no `optional-features.json`, so there is nothing for the build to read.
It sets flags with `.env` or `define`, the same way it sets any other Vite config, and every flag that it does not set gets the default from `ember-source`.
Generators for minimal apps, such as `ember.nvp`, remove the `EmberENV` key from `app/config.ts`, because it does nothing.

### Canary features

Today `isEnabled('FLAG')` reads `FEATURES`, which merges `DEFAULT_FEATURES` with `EmberENV.FEATURES` through `Object.assign`.
The public `FEATURES` export from `@ember/canary-features` is also documented as an object that apps can add to before they create the application.
Each canary feature becomes a `FEATURE_*` constant, with its default next to it, as the read pattern above shows.
`ember-source` code imports `FEATURE_FOO_BAR` and does not call `isEnabled('FOO_BAR')`, because a function call with a string argument does not fold to a constant.
`isEnabled` stays public and reads `ENV.FEATURES`.
A write to `FEATURES` at runtime has no effect on `ember-source`, and development builds throw an error that names the `EMBER_FEATURE_*` flag to set instead.

For beta and release builds of `ember-source`, the ember.js build replaces `import.meta.env?.EMBER_FEATURE_*` with the value that the release channel ships.
Unfinished features are then absent from the published tarball, same as today.

### Optional features

On `main`, `ember-source` only reads one optional feature at runtime: `default-async-observers`, through `ENV._DEFAULT_ASYNC_OBSERVERS`.
The others are either removed ([RFC 704](./0704-deprecate-octane-optional-features.md), [RFC 705](./0705-deprecate-jquery-optional-feature.md)) or have no remaining runtime check.
`EMBER_DEFAULT_ASYNC_OBSERVERS` replaces the runtime read, and `optional-features.json` keeps working through the Vite plugin.

The default for `EMBER_DEFAULT_ASYNC_OBSERVERS` changes from `false` to `true`, because the app blueprint already turns it on.
An app that does not list `default-async-observers` in `optional-features.json` has sync observers today, so for that app the `ember()` plugin defines `false` and behavior does not change.
The `true` default applies to new apps and to builds that do not use the plugin.

> [!NOTE]
> Observers are deprecated ([RFC #1115](https://github.com/emberjs/rfcs/pull/1115)), and v8 removes them.
> `EMBER_DEFAULT_ASYNC_OBSERVERS` goes away with them.

This RFC does not deprecate `@ember/optional-features`.
A later RFC can do that when nothing in `ember-source` reads it.

### `EmberENV`

Each key in `ENV` from `@ember/-internals/environment` gets an `EMBER_*` name from the table rule above, and an initializer that follows the read pattern.

`ember-source` no longer reads `window.EmberENV`.
For most apps this is not visible, because Embroider fills `window.EmberENV` from the same `config/environment.js` that the `ember()` plugin reads.
An app that sets `window.EmberENV` in its own `<script>`, or changes it after the build, loses those values.
Development builds compare `window.EmberENV` with `ENV`, and they throw an error for each key that does not match, with the `EMBER_*` flag to set instead.

### Sveltable deprecations

This RFC does not change how deprecations work.
`DEPRECATIONS`, `deprecateUntil`, and the `isEnabled` and `isRemoved` values stay as they are.

Some deprecations come with a flag that removes the deprecated feature before the major version that removes it.
This RFC calls them sveltable deprecations.
The first one is `deprecate-ember-object` from [RFC 1234](./1234-deprecate-ember-object.md).

Each sveltable deprecation gets a `const` that follows the read pattern.
The name is `EMBER_DEPRECATION_` plus the deprecation `id` in `SCREAMING_SNAKE_CASE`:

```js
export const DEPRECATION_DEPRECATE_EMBER_OBJECT =
  import.meta.env?.EMBER_DEPRECATION_DEPRECATE_EMBER_OBJECT ?? false;
```

When the value is `true`, the deprecated feature is gone, and the minifier removes its code.
The default is `false` until the major version where the deprecation RFC turns the flag on by default.
A deprecation with no flag in its RFC gets no `EMBER_DEPRECATION_*` constant.

### Ecosystem

- Addons: v2 addons can use the same `import.meta.env?.` pattern for their own flags. This RFC does not define a naming rule for them.
- `ember-auto-import`: webpack does not define `import.meta.env` by default. `ember-auto-import` must define the `EMBER_*` values from `config/environment.js` and `config/optional-features.json`. If it does not, classic apps get the defaults and lose their config, so this is a requirement before `ember-source` ships the change.
- Without a bundler, `ember-source` uses the defaults. There is no way to set a flag in that environment.
- SSR and FastBoot: no change. The server bundle gets the same defines as the browser bundle.
- Ember Inspector: no change. It reads `ENV`, which still holds the resolved values.
- TypeScript: `ember-source` declares the `EMBER_*` keys on `ImportMetaEnv`, so the keys autocomplete in `vite.config.mjs` and in app code.
- Blueprints: the app blueprint keeps `config/optional-features.json` and `EmberENV`. No blueprint change is necessary for this RFC.

## How we teach this

The guides page for [feature flags](https://guides.emberjs.com/release/configuring-ember/feature-flags/) and the page for optional features change to show `.env` and `define` first, with `config/environment.js` as the classic option.

The deprecation guides for each sveltable deprecation should document which enviranment variable to set in order to remove the deprecated code.

## Drawbacks

- Apps that set `window.EmberENV` outside of `config/environment.js` must move that config into the build. The error in development builds finds each case, but it is still a change for those apps.
- A flag cannot change at runtime. A test suite that toggles a flag needs one build per value, the same way the ember.js CI runs `ALL_DEPRECATIONS_ENABLED` today (using `define`, you can bring back EmberENV / runtime behavior, however).
- Every supported build must define the `EMBER_*` values if those builds want non-defaults. 
- Without a bundler, there is no way to set a flag.

## Alternatives

- `@embroider/macros` (`getOwnConfig`, `macroCondition`). This works today, but it is Ember-specific and requires Babel. `import.meta.env` is the ecosystem standard, and every modern bundler supports it.
- Do nothing. Apps keep paying for code paths that they turned off.

## Unresolved questions

n/a
