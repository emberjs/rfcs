---
stage: accepted
start-date: 2022-08-26T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/847
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

# Deprecate Support for `ember-cli-qunit` and `ember-cli-mocha` When Generating Test Blueprints

## Summary

This RFC proposes to deprecate support for [`ember-cli-qunit`](https://github.com/ember-cli/ember-cli-qunit) and [`ember-cli-mocha`](https://github.com/ember-cli/ember-cli-mocha) when generating test blueprints.

## Motivation

Both `ember-cli-qunit` and `ember-cli-mocha` were deprecated a long time ago in favor of [`ember-qunit`](https://github.com/emberjs/ember-qunit) and [`ember-mocha`](https://github.com/emberjs/ember-mocha) respectively.

Deprecating (and later on, removing) support for both addons would greatly reduce the amount of test blueprints we have in [`ember-source`](https://github.com/emberjs/ember.js) and [`ember-data`](https://github.com/emberjs/data).

## Transition Path

The `test-framework-detector.js` files [in `ember-source`](https://github.com/emberjs/ember.js/blob/master/blueprints/test-framework-detector.js) and [in `ember-data`](https://github.com/emberjs/data/blob/master/packages/private-build-infra/src/utilities/test-framework-detector.js) will be updated to display a deprecation message whenever `ember-cli-qunit` or `ember-cli-mocha` is detected when generating test blueprints:

```
Support for `ember-cli-qunit` when generating test blueprints has been deprecated.
Please migrate to using `ember-qunit` instead.
```

**Deprecation details:**

| Key     | Value                                            |
| ------- | ------------------------------------------------ |
| `for`   | `'ember-source'`                                 |
| `id`    | `'ember-source.ember-cli-qunit-test-blueprints'` |
| `since` | `{ available: '4.X.X', enabled: '4.X.X' }`       |
| `until` | `'5.0.0'`                                        |

> NOTE: `ember-cli-qunit` and `ember-source` in the deprecation message and details above should be replaced with `ember-cli-mocha` and `ember-data` appropriately.

## Deprecation Guide

### `ember-cli-qunit` Test Blueprints

Support for `ember-cli-qunit` when generating test blueprints has been deprecated.
Please migrate to using `ember-qunit` instead:

**Using Yarn:**

- Run `yarn remove ember-cli-qunit`
- Run `yarn add ember-qunit --dev`
- Update `tests/test-helper.js` to replace any imports from `ember-cli-qunit` with imports from `ember-qunit`

**Using npm:**

- Run `npm uninstall ember-cli-qunit`
- Run `npm install ember-qunit --save-dev`
- Update `tests/test-helper.js` to replace any imports from `ember-cli-qunit` with imports from `ember-qunit`

> NOTE: This guide has been taken from [`ember-cli-qunit`'s `README.md` file](https://github.com/ember-cli/ember-cli-qunit#migrating-to-ember-qunit).

### `ember-cli-mocha` Test Blueprints

Support for `ember-cli-mocha` when generating test blueprints has been deprecated.
Please migrate to using `ember-mocha` instead:

**Using Yarn:**

- Run `yarn remove ember-cli-mocha`
- Run `yarn add ember-mocha --dev`
- Update `tests/test-helper.js` to replace any imports from `ember-cli-mocha` with imports from `ember-mocha`

**Using npm:**

- Run `npm uninstall ember-cli-mocha`
- Run `npm install ember-mocha --save-dev`
- Update `tests/test-helper.js` to replace any imports from `ember-cli-mocha` with imports from `ember-mocha`

> NOTE: This guide has been taken from [`ember-cli-mocha`'s `README.md` file](https://github.com/ember-cli/ember-cli-mocha#migrating-to-ember-mocha).

## How We Teach This

I don't think we need to update any learning material to reflect this deprecation.

## Drawbacks

I don't think there are any real drawbacks, aside from the usual churn that deprecations introduce.

## Alternatives

I don't think there are any meaningful alternatives.

## Unresolved questions

None at the moment.