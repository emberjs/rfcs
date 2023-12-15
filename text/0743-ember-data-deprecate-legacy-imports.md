---
stage: released
start-date: 2021-04-23T00:00:00.000Z
release-date: 2023-09-18T00:00:00.000Z
release-versions:
  ember-data: v5.3.0
teams:
  - data
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/743'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/947'
  released: 'https://github.com/emberjs/rfcs/pull/969'
---

# EmberData | Deprecate Legacy Imports

## Summary

Deprecates `import DS from "ember-data";` and individual imports within the `ember-data` 
package in favor of the imports provided by [RFC#395 packages](https://github.com/emberjs/rfcs/blob/master/text/0395-ember-data-packages.md)

## Motivation

These imports are just a legacy remnant we no longer need.

## Detailed design

Lint rules and a codemod already exist to migrate users to using packages, and have been 
available now for over a year.

The ember-data-packages plugin for ember-cli-babel would be updated so that users would
receive a deprecation when importing from the legacy paths. If required for legacy global
or DS import a runtime deprecation would be added as well.

To resolve, users would need only to run the codemod to convert to using packages.

The deprecation would target `5.0` and would become `enabled` no-sooner than `4.1` (although
it may be made `available` before then).

## How we teach this

Users have already been encouraged to migrate, documentation has already been updated to reflect
the package based imports, and lint rules and codemods already exist. Deprecating and removal at
this point is just a formality as the last stage of fading out the old world.

## Drawbacks

Someone somewhere probably is doing something bad.

## Alternatives

Leave them to waste away.
