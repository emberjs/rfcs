---
stage: accepted
start-date: 2026-04-01T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - cli
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1174
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

# Remove Compat for the Default App Blueprint

## Summary

The default output of `@ember/app-blueprint` should generate an app _without_ `@embroider/compat` and its associated legacy wiring. A new `--compat` flag should be added so that projects which still need v1 addon support, `content-for`, `ember-cli-build`, and the rest of the classic compatibility surface can opt in to it explicitly.

## Motivation

Blueprints set the tone. They're the first thing a new developer sees when they run `npx ember-cli new my-app`, and they're the basis for every tutorial, blog post, and conference talk. When the blueprint ships compatibility layers by default, it signals that _this is how you should build Ember apps_ — even though the compatibility surface exists specifically for migration, not as a recommended long-term architecture.

Today, the default blueprint generates an app that depends on `@embroider/compat`, `ember-cli`, `ember-cli-babel`, `ember-load-initializers`, `ember-resolver`, `@embroider/config-meta-loader`, and several other packages whose sole purpose is bridging old conventions to the new Vite-based build. New apps don't need any of this. They have no v1 addons to support, no legacy build hooks to preserve, no `content-for` templates to fill. The compat layer is dead weight from day one.

Meanwhile, every new app that _starts_ with compat baked in is one more app that will eventually need to _remove_ it. We're creating migration work for projects that never needed the thing being migrated from.

The blueprint should showcase what we want developers to be doing: modern, ESM-native Ember apps built on Vite without legacy compatibility shims. Compatibility is for existing projects that haven't moved yet — and those projects aren't running `ember new`.

> [!NOTE]
> This RFC intentionally does _not_ propose a `--minimal` mode (stripping linting, testing, etc.). That is a separate concern for a future RFC.

### Prior art in this repo

[RFC #1161](https://github.com/emberjs/rfcs/pull/1161) proposed adding `--no-compat` as an opt-in flag alongside `--minimal`. The implementation landed in [ember-cli/ember-app-blueprint#49](https://github.com/ember-cli/ember-app-blueprint/pull/49) but was reverted in [#187](https://github.com/ember-cli/ember-app-blueprint/pull/187) to avoid controversy.

This RFC differs from #1161 in a key way: instead of adding an opt-in `--no-compat` flag (keeping compat as the default), this RFC proposes **flipping the default** so that new apps are generated without compat, and `--compat` becomes the opt-in escape hatch.

## Detailed design

### What changes in the default blueprint output

When a user runs `npx ember-cli new my-app` (without any flags), the generated app should:

**Use ESM conventions:**
- Set `"type": "module"` in `package.json`
- Use subpath imports (`#app/*`, `#config`, `#components/*`) instead of relying on the classic module prefix resolution

**Remove compat-only packages from `package.json`:**
- `@embroider/compat`
- `@embroider/core` (only needed by compat)
- `@embroider/config-meta-loader`
- `@embroider/legacy-inspector-support`
- `@embroider/router` (classic router wiring)
- `ember-cli`
- `ember-cli-babel`
- `ember-load-initializers`
- `ember-resolver` (replaced by the strict resolver from [RFC #1132](https://github.com/emberjs/rfcs/pull/1132))

**Remove compat-only files:**
- `ember-cli-build.mjs` — not needed without the classic build pipeline
- `config/environment.js` — the Node-side config file; replaced by an ESM config module at `app/config/environment.ts` reading from `import.meta.env`

**Simplify HTML entrypoints:**
- `index.html` and `tests/index.html` should directly reference app assets instead of using `content-for` helper blocks and `@embroider/virtual/vendor.js`/`vendor.css`

**Simplify Vite config:**
- Import `ember` from `@embroider/vite` without `classicEmberSupport()`
- No need for `extensions` array that includes `.hbs`

**Simplify `app/app.ts`:**
- Use the strict resolver with `import.meta.glob` ([RFC #1132](https://github.com/emberjs/rfcs/pull/1132)) instead of `compatModules` from `@embroider/virtual/compat-modules`
- No `ember-load-initializers`

**Simplify babel config:**
- Remove `babelCompatSupport()` and `templateCompatSupport()` from `@embroider/compat/babel`

**Simplify test setup:**
- `tests/test-helper.ts` no longer needs compat-specific boot sequence
- `testem.cjs` runs against `dist` directly (via `vite build --mode development && testem ci`)

### The `--compat` flag

Running `npx ember-cli new my-app --compat` generates the same app that the current (pre-RFC) default produces. This flag re-adds all of the compatibility layers listed above.

> [!NOTE]
> `--compat` is not deprecated. It's a supported, first-class mode for projects that need v1 addon compatibility, `content-for` integration, or other classic conventions. The distinction is that it's now _opt-in_ rather than _opt-out_.

### What stays the same

The following are **not** affected by this change:

- Linting, formatting, and testing scaffolding (eslint, prettier, template-lint, qunit, testem) — all remain in the default output
- `@warp-drive/ember` inclusion (follows existing blueprint defaults)
- `.gts`/`.gjs` as the default authoring format (template-tag)
- TypeScript support
- The Vite-based build pipeline itself
- `ember-welcome-page` (follows existing defaults)

### Implementation approach

The implementation mirrors [ember-cli/ember-app-blueprint#49](https://github.com/ember-cli/ember-app-blueprint/pull/49) but inverts the flag direction:

1. The blueprint's `index.js` gains a `--compat` option (boolean, default `false`)
2. Template files use `<% if (compat) { %> ... <% } %>` conditionals for compat-specific content
3. Compat-specific replacement files live under `conditional-files/compat/`
4. When `--compat` is passed, the blueprint includes `ember-cli-build.mjs`, `config/environment.js`, and swaps in compat-aware versions of `app.ts`, `vite.config.mjs`, babel config, and HTML entrypoints
5. `package.json` mutations add compat dependencies when the flag is set

## How we teach this

### For new developers

New developers benefit the most. They see a clean, modern app with fewer files, fewer dependencies, and a simpler mental model. The Guides should be updated to use the no-compat app as the default starting point.

The getting-started tutorial should use the default (no-compat) output. The concepts of `ember-cli-build`, `content-for`, and v1 addons belong in a "Migration" section of the Guides. A separate RFC will address making v2 addons the default and further unrecommending v1 addons.

### For existing developers

A short section in the Guides or a blog post should explain:

- The default output has changed — new apps no longer include compatibility layers
- If you need v1 addon support or classic conventions, pass `--compat`
- Existing apps are not affected. This changes only what `ember new` generates
- If you're starting a new app and aren't sure whether you need `--compat`, you probably don't. If a dependency requires it, you'll know quickly (build errors referencing v1 addon APIs)

For existing apps that want to remove compat, we should publish a "How to remove compat layers from an existing project" guide. The performance gains are substantial — large apps can see up to 70 seconds faster initial boot time after removing all compat layers, depending on their situation.

### Decision guide

| Situation | Recommendation |
|---|---|
| Brand new project, no legacy constraints | `ember new my-app` (default) |
| New project that depends on v1 addons | `ember new my-app --compat` |
| Existing project upgrading | Not affected — continue using your current blueprint |
| Tutorial / demo / blog post | `ember new my-app` (default) |

## Drawbacks

**Some v1 addons are still widely used.** If a new developer picks a popular addon that hasn't shipped a v2 version yet, they'll hit a wall and need to either find an alternative, wait for a v2 release, or restart with `--compat`. This is a real friction point, but it's also pressure on the ecosystem to complete the v2 migration — pressure that the current default actively undermines.

**Two modes means two test matrices.** The blueprint already has conditional code paths (TypeScript vs JavaScript, warp-drive vs not). Adding a compat toggle increases the combinatorial surface. However, the compat path is the _existing, well-tested_ code path — the risk is low.

**Community confusion.** Some developers may not understand why their new app looks different from tutorials written before this change. Clear messaging in the release announcement and updated guides mitigates this.

**Perception of instability.** Changing the default output of `ember new` is visible and can feel disruptive. But the alternative — leaving compat as the default indefinitely — means new apps perpetually start with baggage they don't need, and the ecosystem never gets a clear signal that the migration target is "no compat."

## Alternatives

**Keep `--no-compat` as opt-in (status quo / RFC #1161 approach).** This was tried and reverted. The problem: as an opt-in flag, adoption is slow, documentation defaults to the compat path, and new developers still start with unnecessary complexity. It also means we continue to recommend (by default) a setup we know we want to move away from.

**Provide separate blueprint packages** (e.g. `@ember/app-blueprint-modern`). This fragments the ecosystem and creates a confusing "which blueprint do I use?" decision for new developers. One blueprint with a flag is simpler.

**Wait until all popular addons are v2.** This has been the implicit strategy for years. There will always be one more addon that hasn't migrated. The ecosystem needs a forcing function, and changing the default is that forcing function.

**Deprecate the compat path entirely.** Too aggressive. Existing projects and v1 addons need time. The `--compat` flag gives them that time without holding back new projects.

## Unresolved questions

- **Should we have v2 addon by default first?** No. The no-compat app's design and functionality is "done" and doesn't have to wait for v2 addon by default.
