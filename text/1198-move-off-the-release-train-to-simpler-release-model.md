---
stage: accepted
start-date: 2026-06-05T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1198 # Fill this in with the URL for the Proposal RFC PR
project-link:
suite:
---

# Move off the release train to a simpler release model

## Summary

Ember's "release train" today is really two things bundled together:

1. A six-week release cadence. A new minor ships on a steady, predictable clock.
2. A multi-branch promotion pipeline. Code gets promoted from the default branch through dedicated `beta` and `release` branches (plus LTS branches), shepherded by a small number of folks who manage each release.

This RFC keeps (1) and replaces (2). The six-week cadence stays as it is. What goes away is the extra release branches and the per-cycle process that goes with them.

Concretely:

- Just `main`. No `beta` or `release` branches.
- Stable releases are cut from `main` by [`release-plan`][release-plan] on the same six-week schedule. The `npm publish` is gated behind a protected [GitHub deployment environment][gh-environments], so a maintainer approves before anything publishes.
- We drop the `beta` channel. `@alpha` prerelease is nearly the same as before, but published nightly from `main`, and `ember-source@beta` users either move to `@alpha` or drop the scenario. 

SemVer, the six-week cadence, the deprecation policy, LTS, and the major-version process from [RFC #0830][rfc-830] are all unchanged. The only things that change are the branch structure and the publishing mechanics.

[release-plan]: https://github.com/embroider-build/release-plan
[gh-environments]: https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment
[rfc-830]: https://github.com/emberjs/rfcs/blob/master/text/0830-evolving-embers-major-version-process.md

## Motivation

The expensive part of the release train is all the machinery around it, plus the time it takes maintainers with limited availability to keep it running.

Moving to release-plan also enables us to easily release the other packages in this repo (`@glimmer/syntax`, etc)

### Releases depend on too few people

We don't actually have a formal release-manager rotation. In practice the release comes down to essentially one person per project: @katiegengler for `ember.js`, @mansona for `ember-cli`. This is a bus-factor and burnout risk -- which other teams have already faced. When that person isn't around (due to life, unplanned things, etc (this alone is not a big deal)), the release slips.

Moving off the train means releasing stops being specialized knowledge (despite efforts to write it all down). With [`release-plan`][release-plan], _any_ maintainer can cut a release: they look over the release-preview PR and approve a deployment.

### Extra release branches are overhead

Keeping `beta`, `release`, and the LTS branches alive next to the default branch means constant backporting and branch bookkeeping that only exists to service the shape of the train. A single `main` removes that.

### release-plan automates the mechanical work

Working out the next version number and assembling the changelog are mechanical, error-prone steps that shouldn't be done by hand. release-plan does this for us, deterministically, from the labels and titles of the PRs that got merged. The ecosystem has already standardized on release-plan for almost every addon, so the framework doesn't need a bespoke process that's more work than the tooling everyone else uses.

### Keep the cadence

The six-week cadence is predictable and well understood, so we keep it. We're moving off the branch-and-process model, not the calendar.

## Detailed design

### Branching model

Everything merges to `main`. It's both where work integrates and the source of every release. There are no `beta` or `release` branches, and none of the backporting onto them that the train needs.

### Per-PR release metadata

release-plan uses these labels:

- `breaking` (major impact)
- `enhancement` (minor impact)
- `bug` (patch impact)
- `documentation`, `internal` (recorded, but don't cause a release on their own)

The changelog entry for a release _is_ the set of merged PR titles (you can edit a title later to fix it up), so there's nothing extra to write. release-plan reads the labels and titles of everything merged since the last release, then works out the next version and assembles the changelog. The judgment a maintainer used to apply by reading the diff is captured by the label on each PR when it lands.

### Releasing

The cadence is unchanged. The mechanics are just release-plan's defaults:

1. release-plan keeps a release-preview PR up to date. It bumps the version in `package.json`, edits `CHANGELOG.md`, and records the plan from the labels and titles merged since the last release.
2. On the scheduled six-week date, a maintainer merges that preview PR, which kicks off the publish workflow. (The schedule is the rhythm. Nothing forces a release between scheduled dates, and a date can still be held or moved like it can today.)
3. The publish job targets a protected GitHub environment (say, `npm-publish`) with a required-reviewers rule. The npm token / trusted-publishing identity is scoped to that environment, so nothing publishes until a required reviewer hits Approve on the pending deployment. Maintainers never need npm keys locally.
4. Once approved, the CI job runs `release-plan publish`: it tags, pushes, and publishes to npm (with provenance via OIDC trusted publishing).

Across a cycle, release-plan collapses everything merged into a single version bump (highest impact wins), so six weeks of `enhancement` PRs becomes one minor, exactly like one stable minor per cycle does today. `@alpha` publishes the in-progress version nightly (e.g. `6.5.0-alpha.N`), and the scheduled stable cut publishes the finished version (`6.5.0`) as `@latest`. That stable release is just a snapshot of `main` on the scheduled date, which is the same thing promoting `release` from `beta` produced before.

Approving the deployment is the only manual step left. No manual version edit, no manual changelog, no manual publish, and no branch to cut or promote. Because the gate is a GitHub environment, the normal GitHub permission and audit model applies, so who can approve and who approved what live in repo settings and the audit log.

### Channels

- Stable (`latest`): published from `main` through the gated environment, on the six-week cadence, as above.
- Canary: _unchanged by this RFC._ Canary is the default branch (`main`) consumed from git, and that keeps working exactly like it does today.
- `alpha`: new. A published, npm-installable prerelease of `main`, cut nightly the way Embroider and Glint _used to_ publish theirs. It gives `main` an npm dist-tag (`ember-source@alpha`) for folks who want canary's code without consuming from git. release-plan's defaults handle the version scheme.
- `beta`: removed. The `beta` branch and the `ember-source@beta` dist-tag both go away. Anything that needs extra baking can still merge early and get exercised via canary or `@alpha` before the next scheduled stable release.

### Deprecations, majors, and LTS

Since we keep the six-week cadence, everything layered on top of it is unaffected:

- Deprecations still land as SemVer-minor and get removed in majors, on the same deprecation-freeze schedule.
- Majors aren't a manual process either. A PR gets labeled `breaking`, and because RFC #0830 puts a major on the normal cadence (roughly one major every twelve 6-week minors, after the `M.10` deprecation freeze), `breaking` PRs only merge in that window. So release-plan's label-driven bump produces the major right on schedule, with no manual version work. The major-version process itself doesn't change; it just runs through release-plan from `main`.
- LTS keeps working like it does today.

So this RFC changes _how_ a release is cut and _which branches exist_, not _when_ releases happen or _what_ they guarantee.

### Lockstep across packages

`ember-source` and `ember-cli` keep releasing in lockstep, but lockstep here is a timing thing, not a coupling. Each repo releases independently through release-plan (its natural per-repo mode), and the releases don't need to know about each other. Since both cut stable on the same six-week date, their versions stay lined up. There's no cross-repo coordination step in the workflow; the shared cadence is what keeps them in step.

### What this removes

- The `beta` and `release` branches, and the `ember-source@beta` dist-tag.
- The backporting onto the `beta` and `release` branches.
- The bespoke per-cycle release process. Version bumps, changelog assembly, and the cut/promote steps are all release-plan's job now.

### What this keeps

- The six-week release cadence, unchanged.
- SemVer and every compatibility guarantee we make today.
- The deprecation policy, LTS, and the major-version process (RFC #0830).
- Backports to LTSes, when needed.
- A deliberate human gate before every publish.
- From a consumer's point of view, nothing changes. They won't notice a difference.

## How we teach this

Contributors keep doing what they already do in basically every addon: label each PR with a release-plan label. The PR title becomes the changelog entry.

Maintainers get the ability to release. The role goes from "shepherd the branches and the cut" to "look over the release-preview PR and approve the deployment," which any maintainer can do without npm keys.

`beta` users are the folks whose workflow moves. If your `ember-try` config or CI matrix tests `ember-source@beta`, you remove that scenario, since `@beta` won't exist anymore. You can add `@alpha` if you want a published prerelease. Canary testing (consuming `main` from git) is unaffected.

Docs work:

- Rewrite the website Releases page: `main` is the release line, the six-week cadence is unchanged, the prerelease stream is `@alpha` (published nightly from `main`), and `beta` is gone.
- Update the contributor guide with the labeling workflow.
- Document the environment, who can approve, and how approval works.
- A migration / announcement blog post about dropping `@beta`.

## Drawbacks

- Dropping `@beta` moves some consumers. Any `ember-try` scenario or addon CI matrix pinned to `ember-source@beta` has to remove that scenario (and can adopt `@alpha` if it wants). It's a one-time, mechanical cleanup rather than a lost capability, since canary testing is unaffected, but it's still ecosystem-wide churn that needs coordinating.
- Cultural change. `beta` is a long-standing, load-bearing part of Ember's testing culture and infrastructure, so removing it isn't only a mechanical change.

## Alternatives

- Drop the cadence too (fully continuous releases). A more radical model where every merge can release. We're explicitly _not_ proposing that here, because the six-week cadence works.

## Unresolved questions

n/a
