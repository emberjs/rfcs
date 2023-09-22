---
stage: accepted
start-date: 2023-09-22T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
prs:
  accepted: https://github.com/emberjs/rfcs/pull/966
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

# Remove ember-cli-dependency-checker

## Summary

Modern package managers will report when peers are incorrect, 
which appears to be one of ember-cli-dependency-checker's purposes.
Though ember-cli-dependency-checker also calls out when dep versions are unexpected.

## Motivation

I noticed some erroneous warnings from ember-cli-dependency-checker while
building the app, in particular with dependencies declared with `workspace:^` versions. 

Additionally, fewer dependencies and fewer addons:
- reduce install time
- reduce ember-cli init time (because it scans all v1 addons)

## Detailed design

Remove the dependency from the blueprint.

### Consequences

#### Peers are wrong

Embroider will aggressively warn against incorrect peers.
Strict only.
Example: https://github.com/embroider-build/embroider/issues/1602

Additionally, the modern package managers provide hints about incorrect setups
```
pnpm i # or with --resolution-only

 WARN  Issues with peer dependencies found
apps/repl
├─┬ @ember/test-helpers 3.2.0
│ └── ✕ missing peer ember-source@"*"
```

#### Forget to install dependencies

```
❯ pnpm ember s
Volta error: Could not locate executable `ember` in your project.

Please ensure that all project dependencies are installed with `npm install` or `yarn install`
```

#### Resolution/Overrides collision

This situation occurs when you link to or use `overrides` / `resolutions` to set a version of a dependency that is outside the range of what is specified in the package.json.

There would be nothing to warn of this situation, afaik.

## How we teach this

None of our documentation mentions this dependency, so there is no reason to update any docs.

## Drawbacks

If folks are still using yarn@1, they may run in to issues. Best scenario is to stop using yarn 1.

Plenty of good package management options these days:
- pnpm@8+
- yarn@3+
- npm@9+

## Alternatives

- Leave the incorrect messaging as well as the dep.
- Fix the dependency, keep it.

## Unresolved questions
