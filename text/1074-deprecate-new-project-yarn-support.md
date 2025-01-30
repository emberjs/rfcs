---
stage: accepted
start-date:
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: # update this to the PR that you propose your RFC in
project-link:
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
-->

# Deprecate `--yarn` support for `ember new`

## Summary

Deprecate the `--yarn` flat for `ember new`. We believe newly generated projects
will have a better experience using `npm` or `pnpm` as the package manager.

## Motivation

`ember-cli` currently only supports `yarn@1.x`. The latest version of `yarn` is `4.x`. 
Most contributors to the project are using `npm` or `pnpm` as their package manager and are not motivated 
to update `yarn` support. An [open issue to update support](https://github.com/ember-cli/ember-cli/issues/10339) 
has been open for over nearly 18 months without being addressed. 

We know that modern `yarn` can be used successfully with Ember projects, but it requires
disabling Yarn's `PnP` feature. Generating a new project also requires some gymnastics to
ensure that the correct version of `yarn` is installed.

We believe that the best experience for new projects is to use `npm` or `pnpm` as the package manager.

## Transition Path

Deprecate the `--yarn` flag to `ember new` and `ember addon`. Remove it from help and documentation.

For now, we will continue to support existing projects that use `yarn` as their package manager.


## How We Teach This

Remove any documentation of generating projects with `--yarn`. 

## Drawbacks


## Alternatives

We could update support for `yarn` in the ecosystem, but this would likely only happen if an interested
party were to volunteer. 

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
