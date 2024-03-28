---
stage: accepted
start-date: 2024-03-28T00:00:00.000Z
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
  accepted: https://github.com/emberjs/rfcs/pull/1017/
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

# Update QUnit's Theme 

## Summary

Add [qunit-theme-ember](https://github.com/IgnaceMaes/qunit-theme-ember) to `ember-qunit`, so that the CSS is included by default.

## Motivation

The default qunit theme is very outdated.
Adding a new theme will make testing feel fresh, and more relaxed.

## Detailed design

Finish this PR: https://github.com/emberjs/ember-qunit/pull/1166

Release as a major to be safe, in case folks are already customizing the existing qunit styles.


## How we teach this

Changelog would be fine for ember-qunit.

## Drawbacks


## Alternatives

Leave the theme outdated

## Unresolved questions

n/a
