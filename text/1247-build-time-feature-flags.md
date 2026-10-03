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

`ember-source` reads each flag with the literal expression `import.meta.env?.EMBER_*`:

```js
if (import.meta.env?.EMBER_SYNC_OBSERVERS) {
  // sync observer path
}
```

The bundler replaces the expression with a literal, and the minifier removes the branch.
Each flag defaults to falsy, so an app that sets nothing gets the default behavior.

`EmberENV`, `config/environment.js`, and optional features work the same until v8.
`EMBER_DROP_LEGACY_CONFIG_ENV` removes that code now.
Minimal apps with no config files get a way to set flags for the first time.

A separate RFC will be needed for deprecations of the old styles of flagging.

## Motivation

Today, all of Ember's flags are read at runtime from `window.EmberENV`, so every branch ships to every user:
- an app that turned on `default-async-observers` years ago still ships the sync observer path
- an app that turns off `EmberObject` ([RFC 1234](./1234-deprecate-ember-object.md)) would still ship `EmberObject`

While `ember-source` reads `EmberENV` at runtime, the build cannot remove either branch.
`EMBER_DROP_LEGACY_CONFIG_ENV` removes that read, so `import.meta.env` becomes the one source of truth.

Minimal apps, such as [`v2-app-hello-world-template`](https://github.com/emberjs/ember.js/tree/main/smoke-tests/v2-app-hello-world-template) and the [`ember.nvp`](https://github.com/NullVoxPopuli/ember.nvp) `minimal-app`, have no `config/environment.js` and no `@embroider/virtual/vendor.js`, so they have no way to set a flag at all.

## Detailed design

- Each read is the literal expression `import.meta.env?.EMBER_SOME_FEATURE`. No module re-exports a flag or gives it a second name, so a build tool only needs expression replacement to remove dead code.
- Each flag defaults to falsy. A flag turns on a non-default behavior.
- When a major makes an optional behavior the default, its flag goes away. If the old behavior stays available, it gets a new flag.
- The `?.` is for environments without `import.meta.env` (plain `<script type="module">`, import maps, Node). Those get the default.

Names are `EMBER_` plus `SCREAMING_SNAKE_CASE`.
Values are JSON (`EMBER_RERENDER_LOOP_LIMIT` is a number, for example).

| Flag | Effect |
| --- | --- |
| `EMBER_DROP_LEGACY_CONFIG_ENV` | removes all reads of the legacy config |
| `EMBER_SYNC_OBSERVERS` | observers are sync (async is the default) |
| `EMBER_DROP_DEBUG_RENDER_TREE` | removes the debug render tree, in development too |

### Setting flags

```bash
# .env
EMBER_DROP_DEBUG_RENDER_TREE=true
```

```js
// vite.config.mjs
export default defineConfig({
  define: {
    'import.meta.env.EMBER_DROP_DEBUG_RENDER_TREE': 'true',
  },
});
```

Either one removes the whole debug render tree, in development too.
Today it always ships, because development builds force it on.

Plain Vite gives `.env` values as strings, and the string `"false"` is truthy.
To turn a flag off, leave it out.

Minimal apps have no config files, so they use `.env` or `define` like any other Vite config.

### Legacy config

`window.EmberENV`, `config/environment.js`, `@ember/optional-features`, and `@ember/canary-features` work as they do today until v8.

`EMBER_DROP_LEGACY_CONFIG_ENV` removes the code that reads them.
With the flag set, development builds throw an error for each key in `window.EmberENV`, and the error names the `EMBER_*` flag to use.

### Unstable features

Features in development use `import.meta.env?.UNSTABLE_EMBER_*`.
Code behind an unstable flag can change or break in any release.

### Sveltable deprecations

How deprecations work does not change.

Some deprecations come with a flag that removes the deprecated feature before the major that removes it (for example, `deprecate-ember-object` from [RFC 1234](./1234-deprecate-ember-object.md)):

```js
if (!import.meta.env?.EMBER_DROP_EMBER_OBJECT) {
  // EmberObject
}
```

The flag goes away in the major that removes the code.

### What this replaces

Each deprecation is a follow-up RFC.

| Today | Replacement |
| --- | --- |
| `window.EmberENV`, `EmberENV.FEATURES` | `import.meta.env?.EMBER_*` |
| `@ember/optional-features` | an `EMBER_*` flag per feature |
| canary features | `import.meta.env?.UNSTABLE_EMBER_*` |
| `config/environment.js`, `@embroider/config-meta-loader` | a normal module at `app/config/environment.js` for runtime config |
| `import { DEBUG } from '@glimmer/env'` | export conditions for addons, `import.meta.env.DEV` for apps |

### Ecosystem

- v2 addons can use the same pattern for their own flags.
- TypeScript: `ember-source` declares the `EMBER_*` keys on `ImportMetaEnv`.
- SSR, FastBoot, and the Inspector: no change.

## How we teach this

The [feature flags](https://guides.emberjs.com/release/configuring-ember/feature-flags/) and optional features guides show `.env` and `define` first, and `config/environment.js` as the classic option.

The deprecation guide for each sveltable deprecation documents which environment variable removes the deprecated code.

## Drawbacks

- Apps that set `EMBER_DROP_LEGACY_CONFIG_ENV` must move their `EmberENV` config into the build. The error in development builds finds each key.
- A build that replaces a flag cannot change it at runtime. Vite in development leaves `import.meta.env` as a mutable object, so flags can change there.
- Without a bundler, there is no way to set a flag.

## Alternatives

- `@embroider/macros` (`getOwnConfig`, `macroCondition`). Works today, but it is Ember-specific and requires Babel. `import.meta.env` is the ecosystem standard.
- Do nothing. Apps keep paying for code paths that they turned off.

## Unresolved questions

n/a
