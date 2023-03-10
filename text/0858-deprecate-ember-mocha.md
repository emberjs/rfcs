---
stage: ready-for-release
start-date: 2022-11-08T00:00:00.000Z
release-date:
release-versions:
teams:
  - data
  - framework
  - learning
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/858'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/908'
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

# Deprecate `ember-mocha`

## Summary

This RFC proposes to officially deprecate [`ember-mocha`](https://github.com/emberjs/ember-mocha).

Consequently, this would also deprecate support for `ember-mocha` when 
generating test blueprints. Both `ember-source` and `ember-data` have logic to 
determine the presence of `ember-mocha` in order to use the appropriate test blueprints.

## Motivation

`ember-mocha` has been unmaintained for a while now. The last release was 
published on [Jun 16, 2019](https://github.com/emberjs/ember-mocha/releases/tag/v0.16.0). 
It also seems this release is [not compatible with Ember v4](https://github.com/emberjs/ember-mocha/issues/691). 
Instead of letting `ember-mocha` slowly fade away, it feels better to officially deprecate 
it and clearly communicate this to the community.

Additional benefits:

- Having only `ember-qunit`, saves developers the time and effort of having to make a choice between `ember-mocha` and `ember-qunit`
- The Ember.js ecosystem can fully focus on building functionality around one testing framework

## Transition Path

We should:

- Update `ember-mocha`'s README to state that `ember-mocha` is officially 
deprecated, and that users should consider migrating to [`ember-qunit`](https://github.com/emberjs/ember-qunit) instead
- Officially mark `ember-mocha` as deprecated on [the npm registry](https://www.npmjs.com/)
- Update the test-framework detectors in `ember-source` and `ember-data` to 
deprecate support for `ember-mocha`:
  - https://github.com/emberjs/ember.js/blob/master/blueprints/test-framework-detector.js
  - https://github.com/emberjs/data/blob/master/packages/private-build-infra/src/utilities/test-framework-detector.js
- Write a small guide that explains how to migrate from `ember-mocha` to `ember-qunit`

## Possible Migration Strategy

### 1. Install `ember-qunit` And Its Required Peer Dependencies

Required peer dependencies:

- [`@ember/test-helpers`](https://github.com/emberjs/ember-test-helpers)
- [`qunit`](https://github.com/qunitjs)

Please note that, at the time of writing, `ember-qunit` also requires you to run [`ember-source`](https://github.com/emberjs/ember.js) v3.28 or higher.

Please refer to [the official `app` blueprint](https://github.com/ember-cli/ember-cli/tree/master/blueprints/app/files) for a complete setup of all these packages:

- [Update `tests/test-helper.js`](https://github.com/ember-cli/ember-cli/blob/master/blueprints/app/files/tests/test-helper.ts)
- [Update `tests/helpers/index.js`](https://github.com/ember-cli/ember-cli/blob/master/blueprints/app/files/tests/helpers/index.ts)
- [Update `tests/index.html`](https://github.com/ember-cli/ember-cli/blob/master/blueprints/app/files/tests/index.html#L23-L28)

### 2. Migrate Your Tests One by One

Please refer to [this commit](https://github.com/1024pix/pix/pull/5258/commits/b0eccbdb63caee853d67d2b368c9f6079a334a08) for an exhaustive example on how to migrate your tests.

We also recommend using one of the following codemods, to speed up this process:

- [mocha-to-qunit](https://github.com/freshworks/ember-freshdesk-codemods/tree/master/transforms/mocha-to-qunit) - The original Mocha to QUnit codemod
- [mocha-to-qunit fork](https://github.com/1024pix/ember-codemods/tree/master/transforms/mocha-to-qunit) - Forked from `mocha-to-qunit`, and used in the commit linked above
- [ember-mocha-to-qunit-codemod](https://github.com/yads/ember-mocha-to-qunit-codemod) - A more recently written Mocha to QUnit codemod

### 3. Clean up All References to Mocha

This includes (but not limited to):

- References and packages in your `package.json` file
- References in your `.eslintrc.js` file
- References in custom test blueprints, if you have any
- References in documentation
- ...

### 4. Install `eslint-plugin-qunit` And `qunit-dom`

We also recommend to install and use [`eslint-plugin-qunit`](https://github.com/platinumazure/eslint-plugin-qunit) and [`qunit-dom`](https://github.com/mainmatter/qunit-dom).

- `eslint-plugin-qunit` provides useful ESLint rules for QUnit
- `qunit-dom` provides high-level DOM assertions for QUnit

Though these packages aren't required to complete the migration, they will help you in writing better tests for QUnit. These packages are also part of [the official `app` blueprint](https://github.com/ember-cli/ember-cli/tree/master/blueprints/app/files).

## How We Teach This

References to `ember-mocha` should be removed from all learning materials, for example:

- https://guides.emberjs.com/release/testing/testing-tools/#toc_mocha-chai-dom
- https://guides.emberjs.com/release/testing/testing-tools/#toc_summary
- https://guides.emberjs.com/release/testing/#toc_how-to-filter-tests

## Drawbacks

People using `ember-mocha` will have to migrate to using `ember-qunit` instead 
at some point. This feels like a large migration to take on (depending on project size), 
though I have no experience with this.

## Alternatives

If there is any interest, we could also consider transferring `ember-mocha` to 
the [Adopted Ember Addons](https://github.com/adopted-ember-addons) organisation on GitHub?

## Unresolved questions

None at the moment.