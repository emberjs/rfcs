---
Stage: Accepted
Start Date: 2021-01-04
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember CLI
RFC PR: https://github.com/emberjs/rfcs/pull/696
---

# Replace Travis CI with GitHub Actions in generated Ember CLI projects

## Summary

Travis CI is no longer an easy setup for our users.
GitHub Actions is the obvious replacement.

## Motivation

With the change of ownership, Travis CI is cracking down on open source usage.
https://travis-ci.org has limited the number of builds for open source, which means in practice, most builds are queued for hours.
https://travis-ci.org is also slated for shutdown soon.
https://www.travis-ci.com for open source has been converted to free trial.
After the initial credits are used, you must submit justification for more.
There have been reports that they are refusing requests for more open source credits https://twitter.com/james_hilliard/status/1336081776691843072.

## Detailed design

We will replace the .travis.yml with a .github/workflows/ci.yml.
It will be configured as close as possible to the existing Travis CI setup.
Unlike Travis CI, there is no enable or authorization step.
The presence of a file in .github/workflows/ is automatically recognized and run if hosted on GitHub.

## How we teach this

A blog post could be sufficient explaining this switch.
Existing projects can continue to use Travis CI in its limited capacity.
Those using ember-cli-update could choose to take the new GitHub Actions file or continue using the .travis.yml.
I don't think we cover CI or Travis CI in our docs.

## Drawbacks

Some CI systems are agnostic to code hosting location, but I don't think you can use GitHub Actions without also placing the code there.
This further ingrains the GitHub platform to people that may prefer a different code hosting provider.

GitHub Actions also has a limit how much an open source project can use, but it resets every month.
I haven't heard of anyone hitting the limit though.

GitHub Actions doesn't support retrying individual jobs like Travis CI.
This could impact some user's monthly budget.

## Alternatives

CircleCI and GitLab are other popular alternatives.
Perhaps one would be better as a recommendation from us?

We could also refrain from suggesting/including any CI config files.
