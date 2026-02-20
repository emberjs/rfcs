---
stage: accepted
start-date: 2026-01-05T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - learning
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1160
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

# No Release Train for Blueprints

## Summary

Official Ember blueprint packages (app + addon generation) should not be coupled to the Ember / Ember CLI release train. Instead, blueprint packages should be released continuously from the `main` branch, while `ember-cli` releases pin (exactly) the blueprint versions they bundle for stability. When generating a new app or addon, `ember-cli` should prompt to use either the pinned (bundled) blueprint version or the latest available blueprint version—skipping the prompt when both are the same.

## Motivation

Ember’s release strategy emphasizes stability and predictability: minor releases approximately every six weeks, Long Term Support (LTS) for 54 weeks, and a commitment to SemVer and backwards compatibility. (See https://emberjs.com/releases/.)

However, default blueprints (what `ember new` / `ember addon` generate) have different ergonomics than framework runtime code:

- Blueprint changes primarily affect *newly generated* projects, not existing ones.
- The common social workflow is to propose blueprint improvements via PRs to the default branch.
- When blueprint improvements are tied to a release train, users can wait weeks (or longer) to see improvements in newly generated apps and addons.

Additionally, while blueprints have historically been used as part of Ember’s upgrade and migration story, that can have an unintended cost for *new* users: improvements that primarily exist to support long-lived apps and slower-moving upgrade paths can accumulate in the defaults. This can leave new users feeling like they are “paying” for technical debt created by projects that cannot keep up, and that the defaults are holding them back.

We can also think of official blueprints as a *product*: they are the first experience many users have with Ember, and they encode decisions about tooling, project layout, and “pit of success” defaults. Like any product, we want customers to receive fixes and refinements as quickly as possible.

We want a workflow where blueprint changes can ship as soon as they are merged, while still preserving the stability guarantees that come from Ember CLI releases.

## Detailed design

### Goals

- Blueprint packages ship continuously from `main`.
- `ember-cli` defaults to a known-good, pinned blueprint version.
- Users may choose the latest blueprint version at generation time.
- If pinned and latest match, do not prompt.

### Non-goals

- This does not change Ember’s runtime SemVer or deprecation policies.
- This does not change existing apps/addons after generation.

### Packaging and distribution model

Official blueprints already live in independently versioned npm packages. This RFC proposes changing how those *official blueprint packages* are released and consumed, not physically relocating the code.

Minimal model:

- One package for app generation and one for addon generation.
- Each package is published from `main` upon merge (or via a release workflow that tracks `main`), with SemVer versioning.

This RFC intentionally does not prescribe exact package names, but the model assumes:

- The packages are owned/maintained by the Ember CLI team.
- Publishing is automated and reproducible.

### Official blueprint release process

The release process for official blueprint packages should be decoupled from the `ember-cli` release train:

- Merges to `main` can result in a new blueprint package release (with appropriate SemVer discipline).
- `ember-cli` continues to have its own release cadence, and periodically updates the pinned blueprint versions it bundles.

This makes blueprint improvements available to users quickly (via “latest”), without changing the default behavior of any specific `ember-cli` release.

### Pinning for stability

Each `ember-cli` release must pin to an *exact* blueprint version for each blueprint package it uses.

In practice, this means the `ember-cli` `package.json` uses exact versions (e.g. `"1.2.3"`, not `"^1.2.3"` or `"~1.2.3"`) for blueprint dependencies.

Rationale:

- It ensures that `ember-cli@X` always generates the same output by default.
- It keeps the “known-good” blueprint version aligned with the `ember-cli` release QA surface.
- It avoids accidental behavior changes due to dependency resolution.

### Generation-time prompt

When a user runs `ember new` or `ember addon`, `ember-cli` must:

1. Determine the bundled blueprint version (from its pinned dependency).
2. Check the registry for the latest blueprint version.
3. If the versions differ, prompt the user:

  - Use bundled version (recommended): generate with the pinned version.
  - Use latest version: generate with the latest published blueprint version.

4. If the versions are the same, proceed with no prompt.

#### Prompt text and behavior

The prompt should be explicit about the tradeoff:

- Bundled: maximizes stability and matches the `ember-cli` release.
- Latest: includes the newest blueprint improvements.

If the registry lookup fails (offline, network errors, etc.), `ember-cli` should proceed with the bundled version and must not block generation.

#### Examples

If the bundled blueprint version and latest blueprint version differ:

- `ember new my-app`
  - Prompt: Use bundled version (vX.Y.Z, recommended) / Use latest version (vX.Y.Z)

If the bundled blueprint version and latest blueprint version are the same:

- `ember new my-app`
  - No prompt; generation proceeds.

If the user is offline or the registry is unavailable:

- `ember new my-app`
  - No prompt; generation proceeds using the bundled version.

#### Non-interactive environments

When `ember-cli` is running in a non-interactive environment (e.g. CI), the default must be bundled to preserve determinism.

This RFC recommends (but does not require) an explicit opt-in flag to avoid relying on interactive prompting in automation:

- `--use-latest-blueprints` (or equivalent) to force using the latest blueprint version.

### Implementation notes (non-normative)

- Blueprint packages can be installed into the project being generated (or into a temporary location) and executed similarly to how built-in blueprints work today.
- Caching the “latest version” check for the duration of a command is acceptable; it must not change the pinned default.

### Rollout and work remaining

This RFC describes an end-state, but acknowledges that we have work to do to get there.

In particular, the official app blueprint has historically served both as a “starting point” and as part of the compatibility/upgrade story. The `@ember/app-blueprint` package only very recently (as of writing this RFC: 6.10-alpha) added a `--no-compat` option, which is a promising step toward a clearer separation between “new app defaults” and “compatibility scaffolding”, but we should expect incremental iteration before the blueprint defaults and teaching story fully reflect the goals described above.

### Compatibility and support expectations

Blueprint packages should follow the same general compatibility expectations as Ember CLI:

- Changes should be backwards-compatible when feasible.
- When a breaking change is necessary, it should be done via a SemVer major release of the blueprint package.

Because `ember-cli` pins versions, breaking blueprint changes only affect users who explicitly choose the latest version (or explicitly opt into the newer major), and will not silently affect users using the bundled version.

### Why this simplifies releases

With this design:

- Blueprint fixes and improvements can ship as soon as they are merged to `main` (via blueprint package releases).
- Ember CLI releases no longer serve as the primary vehicle for blueprint delivery; they only select which blueprint versions are bundled.
- Users who want the newest generator output don’t have to wait 3 months for the next `ember-cli` release.

## How we teach this

Update the Ember CLI documentation and “Creating an app” onboarding path to describe:

- That `ember-cli` ships with bundled blueprint versions (stable default).
- That users may opt into the latest blueprint versions at generation time.
- What the prompt means, and when they will or won’t see it.

Suggested teaching framing:

- “Ember’s release train remains the source of stability for runtime behavior.”
- “Blueprints are project templates: they can evolve faster without affecting existing apps.”
- “Official blueprints show how you *should* build a new Ember app or addon today.”

This RFC is intentionally scoped to blueprint release and consumption mechanics. A separate RFC should address the broader product/education distinction between upgrading an existing application and starting a new application, including what guarantees we want to make to each audience.

If this RFC is accepted, we should also document recommended contributor workflow:

- PRs to blueprint repositories/packages land on `main` and release continuously.
- `ember-cli` periodically updates its pinned blueprint versions.

## Drawbacks

- Pinning a blueprint version in ember-cli means we can't ship bugfixes to the blueprints without an ember-cli release. I think this is a good tradeoff

## Alternatives

- Keep blueprints coupled to `ember-cli` releases. This preserves the status quo and its complexity and retains the long delay between blueprint improvements and user adoption.
- Provide a nightly/preview blueprint channel. Adds complexity and still requires decisions about stability vs freshness.
  - We already have workflows in our ecosystem that do "unstable" releases on main, independent of release

## Unresolved questions

n/a