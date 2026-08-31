---
stage: accepted
start-date: 2026-01-12T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1161
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

<!-- Replace "RFC title" with the title of your RFC -->

# Add `--no-compat` and `--minimal` modes to the Ember app blueprint

## Summary

This RFC proposes standardizing and supporting two new opt-in modes when generating apps from the Ember app blueprint (`@ember/app-blueprint`): `--no-compat` and `--minimal`.

`--no-compat` generates an Ember app that intentionally omits legacy compatibility layers needed for older addon formats and older build/boot conventions. It prioritizes modern ESM tooling, significantly faster installs/builds/boot, and smoother integration with the broader Vite/ESM ecosystem.

`--minimal` builds on `--no-compat` and additionally strips “project hygiene” tooling (linting/formatting/testing scaffolding), producing a small app shell well-suited for demos and for embedding Ember in other repositories and multi-framework environments (e.g. icon libraries, Vite plugin repos, Astro integrations).

This RFC also documents the project intent to make “no-compat” the default for newly generated apps in a future major step, after ecosystem readiness and a clear transition path.

## Motivation

Ember’s “new app” experience increasingly targets modern JavaScript workflows: ESM-first packages, Vite-powered builds, and native browser capabilities. However, today’s default generated app must carry a set of compatibility layers to support:

- older addon formats (notably v1 addons)
- legacy build/boot wiring
- legacy templating and HTML “content-for” integration points
- Node-side configuration files and conventions

Those layers are valuable for upgrading existing apps and for broad compatibility, but they impose real costs for brand-new apps:

- Slower installs and heavier dependency trees (e.g. `ember-cli` and its transitive deps)
- Slower build and boot, especially in large apps
- Friction integrating Ember into “ESM-native” ecosystems (Vite SSR, Astro, etc.)

We need first-class support for two distinct “new app” use-cases:

1. **Modern Ember apps that do not need legacy compatibility**
   - Apps that do not depend on v1 addons
   - Apps that do not need HTML `content-for`
   - Apps that want ESM-first conventions like `"type": "module"`

2. **Ultra-light Ember shells for demos and integration**
   - Minimal “it boots” projects for embedding Ember into other repos
   - Multi-framework demo matrices (React/Vue/Svelte/Ember) where the repo uses a shared test harness and linting strategy
   - Integration spikes in other ecosystems (e.g. Iconify, Vite plugin repos, Astro integrations), where the generated app is a fixture rather than a long-lived production app

This RFC formalizes the public API surface introduced by these flags and sets expectations around support, documentation, and the migration path toward a future where “no-compat” can become the default.

## Detailed design

### Scope

This RFC covers **the behavior and support commitments** for two generator flags on the Ember app blueprint, as implemented in `ember-cli/ember-app-blueprint` PR #49:

- `--no-compat`
- `--minimal`

This RFC does **not** propose removing existing defaults today. The default remains the “compat” behavior. Instead, we:

- document what these modes generate
- define the supported/unsupported compatibility boundaries
- describe the intent and prerequisites for making “no-compat” the default in the future

### Terms

- **compat mode**: the current default generated output which includes legacy compatibility support (notably `@embroider/compat` and the conventions it enables).
- **no-compat mode**: an opt-in generated output with the compatibility layer removed.
- **minimal mode**: an opt-in output built on no-compat, with additional removal of linting/formatting/testing scaffolding.

### CLI surface area

When generating a new app from `@ember/app-blueprint`, users may pass:

- `--no-compat` (or equivalently `--compat=false`)
- `--minimal`

Precedence rules:

- `--minimal` **forces** `no-compat` behavior.
- If `--minimal` is set, explicit `--compat`/`--no-compat` flags are ignored.

### Behavioral summary

#### `--no-compat`

When `--no-compat` is enabled, the generated app prioritizes ESM and modern tooling and removes older compatibility features.

It includes the following high-level changes (as in PR #49):

- **ESM by default**
  - `package.json` includes `"type": "module"`
  - Adds an `imports` map for ergonomic subpath imports (e.g. `#app/*`, `#config`, `#components/*`).

- **Node tooling expectations updated**
  - `package.json` sets a modern minimum Node version (in PR #49: `engines.node: ">= 24"`).

- **Remove legacy compatibility dependencies**
  - Removes `@embroider/compat` and related compat-only dependencies.
  - Removes `ember-cli` from the generated app’s devDependencies.
    - Tradeoff: the generated project no longer includes local scaffolding via `ember g`.
    - Mitigation: users can still run generators via `pnpm dlx ember-cli g ...` / `npx ember-cli g ...` when needed.

- **Remove legacy features supported via compat**
  - No v1 addon compatibility (the generated app assumes v2+ addons).
  - No `.hbs` file support for components/tests (template compilation is handled via modern compilation and template-tag usage).
  - No HTML `content-for` usage in HTML entrypoints.
  - No Node-side `config/environment.js`-style “node-land config”. Instead, config is generated as an ESM module (in PR #49: `app/config/environment.ts`) which reads from `import.meta.env`.

- **Vite integration changes**
  - Vite config conditionally omits classic support when in no-compat mode.

- **HTML entrypoints adjusted**
  - `index.html` and `tests/index.html` are made conditional:
    - In compat mode, they continue using `{{content-for ...}}` and Embroider virtual CSS/JS.
    - In no-compat mode, they instead directly link app styles and omit `content-for` blocks.

- **Testing harness adjustments** (when tests are present)
  - The generated `config/environment` exports an `enterTestMode()` helper in non-minimal builds.
  - `tests/test-helper` calls `enterTestMode()` in no-compat builds.
  - `testem` is configured to run from `dist` in no-compat mode.

#### `--minimal`

`--minimal` is explicitly intended for **demos** and **integration fixtures**.

It includes everything from `--no-compat` and additionally:

- **Removes linting/formatting/testing scaffolding**
  - Strips common `lint:*` and `format` scripts.
  - Removes common lint/format/test devDependencies (eslint, prettier, ember-template-lint, qunit/testem, etc.).

- **Inverts a couple of defaults to reduce “stuff” in the minimal app**
  - Data layer defaults are more conservative:
    - warp-drive becomes opt-in (`--warp-drive`), rather than something you later remove.
  - “Welcome page” becomes opt-in (`--welcome`).

- **Produces a very small runtime surface**
  - Uses a strict application resolver oriented around explicit module discovery.
  - Uses `import.meta.glob` to eagerly load key module categories.

In other words:

- `--minimal` implies `--no-compat`.
- `--no-compat` does not imply `--minimal`.

### Intent: making no-compat the default (future)

This RFC records the intent to eventually make “no-compat” the default for newly generated apps.

That shift is **not** proposed to happen immediately. It requires:

- Addon ecosystem readiness (v2+ coverage for common needs)
- Clear messaging for new users about what is/isn’t supported
- A clear story for teams that still need compatibility (explicit `--compat` mode)

The long-term goal is:

- **Default**: no-compat (modern, faster, ESM-first)
- **Opt-in**: compat mode for projects that need legacy support as long as the slower/long-tail upgrade process from existing compat-using projects
- **Opt-in**: minimal mode for fixtures, demos, and embedded use-cases

### Ecosystem and tooling implications

#### Lint rules

- `--no-compat` should continue to ship with lint/test defaults unless explicitly requested otherwise.
- `--minimal` intentionally removes lint/test tooling, so no lint rules should be assumed.

#### Ember Inspector / debuggability

No-compat does not inherently reduce debuggability, but removing legacy tooling changes how apps are booted and where configuration comes from. Documentation and troubleshooting guides should be updated accordingly.

#### SSR / Vite SSR

The `type=module` default and the removal of legacy compat layers are aligned with Vite SSR and other ESM-first tooling. This RFC does not mandate a particular SSR solution but aims to remove common blockers.

#### Addon ecosystem

`--no-compat` draws a bright line: v1 addons are not supported in the generated project.

This is both a feature (simpler, faster, modern) and a risk (some users will hit unsupported addons). The default remains compat for now to mitigate this.

#### IDE support

ESM + imports maps can improve editor navigation (e.g. `#app/*`), but requires that tooling understands `imports`. The generated output should remain aligned with common TypeScript/JS tooling.

#### Blueprints and generators

Because `ember-cli` is removed in `--no-compat`, generator UX changes. The recommended story is:

- use `npx ember-cli g ...` / `pnpm dlx ember-cli g ...` when you need scaffolding
- keep the generated project itself light by default

## How we teach this

We should teach this as a **mode switch for the generated app**, not as a new “framework feature”.

Key messaging:

- `--no-compat` is for **modern apps** that don’t need legacy addon/build compatibility.
- `--minimal` is for **fixtures, demos, and embedding** Ember in other repos.

Concrete teaching changes:

1. Update the app blueprint README and Ember CLI docs to include an “Options” section describing:
   - what each flag does
   - what you give up
   - who should use it

2. Provide a small decision table:
   - “Are you new to Ember?” → start with default (no-compat)
   - “Do you need v1 addons?” → default/compat
   - “Are you building an ESM-first app or want the fastest build/boot?” → `--no-compat`
   - “Are you embedding Ember into another repo?” → `--minimal`

3. Provide a follow-up guide: “From minimal to production”
   - how to add linting
   - how to add testing
   - how to add routing/UI/etc.

4. Call out the future direction explicitly:
   - We intend “no-compat by default” in the future.
   - `--minimal` remains a niche but valuable tool for integration and demos.

## Drawbacks

- **More public API surface area**: every new flag is a support commitment.
- **Potential confusion for new users**:
  - users may pick `--minimal` and be surprised there are no tests/lints.
  - users may pick `--no-compat` and be surprised a v1 addon doesn’t work.
- **Ecosystem fragmentation risk**: “compat” vs “no-compat” could lead to duplicated docs and support channels.
- **Node version constraints**: setting a high minimum Node version may be a barrier in some environments.
- **Generator/scaffolding UX tradeoff**: removing local `ember-cli` makes the generated app lighter, but removes the familiar `ember g` workflow unless invoked via `npx/pnpm dlx`.

## Alternatives

1. **Separate blueprints instead of flags**
   - e.g. publish `@ember/app-blueprint-no-compat` and `@ember/app-blueprint-minimal`.
   - Pros: fewer flags; clearer separation.
   - Cons: more packages to discover/version; harder to share fixes across variants.

2. **A single `--mode=` flag**
   - e.g. `--mode=compat|no-compat|minimal`.
   - Pros: explicit, scales to new modes.
   - Cons: breaks existing expectations; adds parsing/UX complexity.

3. **Keep everything as-is**
   - Don’t add new modes.
   - Cost: continued friction for modern apps and for embedding Ember in other ecosystems; continued overhead for users who don’t need legacy compatibility, will result in forks of blueprinting entirely.

4. **Make no-compat the default immediately**
   - Pros: forces ecosystem to modernize.
   - Cons: too disruptive; poor onboarding experience until addons/docs are ready.

## Unresolved questions

n/a