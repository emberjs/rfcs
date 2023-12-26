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

# Deprecate named inject 

## Summary

As of `ember-source@4.1` (and [RFC#752](https://github.com/emberjs/rfcs/pull/752)),  `inject` is an old alias that's no longer needed

## Motivation

`import { service } from '@ember/service'`
makes more sense than 
`import { inject as service } from '@ember/service'`

This allows us to slim down our public API surface area to more of _what's needed_.


## Transition Path

Most folks can do a mass find and replace switch from `inject as service` to just `service`.

## How We Teach This

The docs / guides already use the new import path.

## Drawbacks

n/a

## Alternatives

n/a

## Unresolved questions

n/a
