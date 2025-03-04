---
stage: accepted
start-date: 2023-11-02T16:05:11.000Z 
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/985
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

# Change the default addon blueprint

## Summary

`@embroider/addon-blueprint` has been in progress for a while, and thorough testing and usage, so it should become the default addon blueprint, replacing the existing addon blueprint in ember-cli (a the default)


## Motivation

We want to encourage folks to use v2 addons instead of the classic addon blueprint.

V1 addons are built by apps on every boot, every build.
V2 addons are built at publish time, so the app can build faster.

Even though we still have more improvements planned for the v2 addon blueprint, there are _only disadvantages_ to keeping the v1 addon blueprint the default. 

When we make changes to the v2 addon blueprint, we the next version of ember-cli will get those changes without needing to change anything about `ember-cli`'s addon-support.

## Detailed design

`@embroider/addon-blueprint` will remain its own repo. `ember addon` will use this by default.

When `ember-cli` is published, it will lock in the version of `@embroider/addon-blueprint` at the time of publish. For example, if `@embroider/addon-blueprint` is version "2.3", when `ember-cli` 6.1 is published, if someone is using `ember-cli` 6.1, they'll always run the "2.3" version of `@embroider/addon-blueprint` -- the equivelent of doing `--blueprint @embroider/addon-blueprint@2.3` today. 

Folks can still use the "latest" version of `@embroider/addon-blueprint` via specifying the `--blueprint "@embroider/addon-blueprint` argument and value.

## How we teach this

We already don't have much documentation for authoring addons -- and at this point, we probably have more v2 addon documentation than we do v1 addon documentation.

As a part of making v2 addons default, we will need to provide actual documentation and display on the ember-guides
- https://github.com/embroider-build/embroider/blob/main/docs/v2-faq.md 
- https://github.com/embroider-build/embroider/blob/main/docs/addon-author-guide.md
- https://github.com/embroider-build/embroider/blob/main/docs/porting-addons-to-v2.md  
- https://github.com/embroider-build/embroider/blob/main/docs/peer-dependency-resolution-issues.md  
- https://github.com/embroider-build/addon-blueprint?tab=readme-ov-file#embroideraddon-blueprint
  - the README here in particular suggests some extra options that would be helpful to have first-class support in `ember-cli`


## Drawbacks

The tooling team wants to make the "non-monorepo" version of the V2 Addon blueprint the default in `@embroider/addon-blueprint` -- but this is a separate change from making the `@embroider/addon-blueprint` the default in `ember-cli`, as development efforts there would be solely in `@embroider/addon-blueprint` (and supporting packages). During this time, folks could feel like there is churn in the blueprint setup.
For libraries (and specifically in our v2 addon documentation), we need to make it clear that there are many paths to emitting valid JavaScript for publishing, and you don't need to update anything if the output of your project (and way it is tested) is still sufficient.

## Alternatives

- do nothing (tho v1 addons are harmful for app speed)

## Unresolved questions

TBD?
