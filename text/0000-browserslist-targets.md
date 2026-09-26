---
stage: accepted
start-date: 2026-09-25T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - learning
prs:
  accepted:
project-link:
suite:
---

# Browser targets from the browserslist config

## Summary

New apps set their browser targets with the standard browserslist config (the `browserslist` key in `package.json`) instead of `config/targets.js`. The build reads it, and `config/targets.js` keeps working but is discouraged.

## Motivation

`config/targets.js` is an Ember-only file that holds a browserslist query. Other tools, such as `@babel/preset-env`, Autoprefixer and `eslint-plugin-compat`, already read the standard browserslist config. Apps that use them end up with two lists that can drift apart.

Using the standard config gives every tool one list, and removes one Ember-specific file from new apps.

## Detailed design

### Blueprint

`@ember/app-blueprint` stops generating `config/targets.js` and puts the same queries in `package.json` ([ember-app-blueprint#293](https://github.com/ember-cli/ember-app-blueprint/pull/293)):

```json
"browserslist": [
  "last 1 Chrome versions",
  "last 1 Firefox versions",
  "last 1 Safari versions"
]
```

### Where targets come from

`@embroider/vite` sets Vite's `build.target` ([embroider#2820](https://github.com/embroider-build/embroider/pull/2820)). ember-cli's `Project#targets`, which `ember-cli-babel` uses for v1 addons, feeds Babel ([ember-cli#11066](https://github.com/ember-cli/ember-cli/pull/11066)). Both look for targets in this order:

1. `config/targets.js`, if present. This is unchanged. `@embroider/vite` also requires it to export `browsers`, as it does today.
2. Otherwise the browserslist config, as found by `browserslist.loadConfig()` from the project root: `package.json#browserslist`, `.browserslistrc`, or the `BROWSERSLIST` environment variable.
3. Otherwise the current defaults. That's Vite's default target in `@embroider/vite`, and `last 1 Chrome/Firefox/Safari versions` in ember-cli.

`config/targets.js` wins when both exist. An app that customized `config/targets.js` and upgrades with `ember-cli-update` can end up with both: the blueprint's default list in `package.json` and its own list in `config/targets.js`. If browserslist won, the app's own list would be silently replaced.

### Messaging

Whenever `@embroider/vite` uses `config/targets.js`, it prints a warning. The warning says to move the list to `package.json#browserslist` and delete `config/targets.js`. When both exist, it also says which one is in use.

### Later

A follow-up deprecation RFC can flip the order or drop support for `config/targets.js` in a future major, once the warning has been out for a while.

## How we teach this

The Guides page [Build Targets](https://guides.emberjs.com/release/configuring-ember/build-targets/) switches from `config/targets.js` to the `browserslist` key in `package.json`. It links to the browserslist docs for queries and other config locations. It also gets a short note for existing apps: move the list and delete `config/targets.js`.

## Drawbacks

- There are two supported ways to set targets until the deprecation lands.
- Apps that keep `config/targets.js` see the warning on every build.
- `config/targets.js` can compute its list in JavaScript. Browserslist covers the common case with environment sections (`production`, `development`, picked by `BROWSERSLIST_ENV` or `NODE_ENV`), but not arbitrary logic. Apps that need more can set the `BROWSERSLIST` environment variable, or keep `config/targets.js` until the deprecation.

## Alternatives

- **Browserslist wins when both exist.** The rule is simpler and closer to the end state. But it silently changes targets for apps that end up with both after an upgrade, and it changes what the public `Project#targets` returns.
- **An `eslint-plugin-ember` rule against `config/targets.js`.** Lint rules check file contents, not whether a file exists. The build warning reaches the same people with no setup.
- **Do nothing.** New apps keep an Ember-specific file that duplicates a standard config.

## Unresolved questions

- When to deprecate `config/targets.js`, and whether the deprecation flips the order or removes support.
- Should classic (non-Vite) builds print the same warning? Today only `@embroider/vite` does.
