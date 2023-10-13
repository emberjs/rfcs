---
stage: released
start-date: 2023-03-25T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - learning
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/918'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/954'
  released: 'https://github.com/emberjs/rfcs/pull/978'
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

# Deprecate Support for Travis CI

## Summary

This RFC proposes to officially deprecate support for generating a Travis CI 
config file when creating a new app or addon.

## Motivation

Since Travis CI announced the end of its unlimited support for open-source 
projects, most of the (the entire?) Ember community has switched over to using
GitHub Actions instead. This basically leaves the `.travis.yml` files in the 
`app` and `addon` blueprints unused. Even though the maintenance cost of keeping 
these files around is pretty low, not having to maintain them would be even 
better. It would make [PRs like this](https://github.com/ember-cli/ember-cli/pull/10222) 
slightly less cumbersome. Also, since almost no one actually uses these files, 
it becomes harder to know/ensure they are up to date and follow the current best 
practices.

## Transition Path

We should:

- Show a deprecation warning when creating a new app or addon using the 
`--ci-provider=travis` option
- Show a deprecation warning when picking the `Travis CI` option during the 
interactive new flow
- Add a comment to the `.travis.yml` files in the `app` and `addon` blueprints 
mentioning that they are deprecated - Adding a comment is the easiest thing to 
do implementation wise and people who _do_ wish to continue using Travis CI, can 
simply remove the comment again

## How We Teach This

I _think_ we would only need to remove all references to Travis CI from the 
learning materials.

## Drawbacks

Can't think of any at the moment.

## Alternatives

Continue supporting Travis CI.

## Unresolved questions

None at the moment.