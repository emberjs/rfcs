# Improved Release Process

- Start Date: 2016-03-26
- RFC PR: #46
 
## Summary & Motivation

ember-cli has followed an ad hoc release process throughout its existence which has made it difficult to know exactly when releases would come out, what features would and would not be supported, and the degree to which it would support existing Ember applications. With the proposal for lockstep SemVer there were ideals of guaranteeing compatibility, which we have mostly met, but that resulted in making decisions of delaying an official 2.X release of ember-cli to avoid additional major version bumps.

We propose that we adopt a pattern similar to Ember itself in order to align with the expectations of the Ember community, more-clearly communicate around our release lifecycle, and provide rigor around our support structure. Anything which is not specifically called out as a difference in this document is inferred to be following the patterns specified by Ember itself.

## Channel Design

To begin there will be three separate channels: canary, beta, and release. We intend to investigate an LTS channel after this process has matured.

- **Canary**: represents the latest work in ember-cli, and is synonymous with the `HEAD` of the `master` branch and is the least stable of all channels.
- **Beta**: branched off of master every six weeks, exact commit decided upon manually. Updated and released weekly with commits that are prefixed `[BUGFIX beta]`. Less stable than `release` as it is a proving ground. No new features will be added once the branch has been created to allow for existing features to mature. Tags will match Ember's patterns, for example `v2.6.0-beta.1`. Branch name: `beta`.
- **Release**: branched off of Beta every six weeks. Only rarely will this be updated, but possible for security issues and uncaught regressions. Branch name: `release`.
 
ember-cli will not support daily releases as time-based packaging doesn't make a lot of sense.

## New Features

New features to ember-cli must be protected by feature flags. Incomplete and WIP features will be available in the Canary channel, but will not be available in the Beta or release channels.

## Tooling Design

We must create additional tooling and patterns in order to make this efficient. Since ember-cli successfully installs and works from npm without modification we don't need to bundle and publish an asset for each Canary build. We'll publish tags to npm for `beta` and `release` channel releases so that they're not tied to a git remote URL. The `latest` tag for npm (the default when installing via `npm install -g ember-cli`) will track our `release` channel at all times. We will publish tagged releases (i.e. v2.6.0-beta.1) to the npm `beta` tag which is used via `npm install --save-dev ember-cli@beta`.

## Timeline

Since the ember-cli project is presently designed to track Ember development, we'll run our release schedule on a one week delay from Ember itself. This ensures that we're able to incorporate the latest changes from Ember into ember-cli and gives us a week to check for Ember-introduced regressions. As Ember itself becomes an npm module this will become less of a concern and we can diverge on our release schedule as best suits the ember-cli project. We will ship the last beta coincidentally with the newest Ember release.

## Drawbacks

The largest drawback is also a feature: we require more rigor in our release processes. This process presently requires a weekly manual review of new commits to master and their prefixes which then get cherry-picked to the appropriate `release` and `beta` branches.

We've also encountered issues with `npm` in the past which may require investigation into other tools.

## Effort

In order to undertake this task, there are multiple workflows which must occur:

- [ ] Updates to the website and documentation communicating this plan.
- [ ] Teaching new patterns to ember-cli contributors, most specifically commit tagging and feature flagging.
- [ ] Increased automation of the release process.
- [ ] Tooling to support feature flags.
 
## References

- [Ember's Post-1.0 Release Cycle](http://emberjs.com/blog/2013/09/06/new-ember-release-process.html)
- [Ember RFC #56 - Improved Release Cycle](https://github.com/emberjs/rfcs/blob/master/text/0056-improved-release-cycle.md)
- [Announcing Ember's First LTS Release](http://emberjs.com/blog/2016/02/25/announcing-embers-first-lts.html)
- [ember-cli Release Instructions](https://github.com/ember-cli/ember-cli/blob/master/RELEASE.md)
 