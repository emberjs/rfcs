---
stage: accepted # FIXME: This may be a further stage
start-date: 2022-02-25T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
prs:
  accepted: https://github.com/emberjs/rfcs/pull/801
project-link:
---

# Deprecate `blacklist` and `whitelist` build options

## Summary

This RFC proposes to deprecate the `blacklist` and `whitelist` build options, in
favour of the `exclude` and `include` build options.

## Motivation

In [RFC 639](https://emberjs.github.io/rfcs/0639-replace-blacklist-whitelist.html),
the `exclude` and `include` build options were introduced. These options provide
exactly the same functionality as the `blacklist` and `whitelist` build options,
but these new terms are more neutral.

RFC 639 was implemented and released in [Ember CLI v4](https://github.com/ember-cli/ember-cli/blob/master/CHANGELOG.md#v400),
which means we should be able to deprecate the old terms in favour of the new
ones somewhere in the near future.

## Transition Path

When either the `blacklist` or the `whitelist` build option is used, the
following deprecation message will be triggered:

```
Using the `addons.blacklist` or the `addons.whitelist` build option is deprecated.
Please use `addons.exclude` or `addons.include` respectively instead.
```

**Deprecation details:**

| Key     | Value                                           |
| ------- | ----------------------------------------------- |
| `for`   | `'ember-cli'`                                   |
| `id`    | `'ember-cli.blacklist-whitelist-build-options'` |
| `since` | `{ available: '4.X.X', enabled: '4.X.X' }`      |
| `until` | `'5.0.0'`                                       |

## Deprecation Guide

The `addons.blacklist` and the `addons.whitelist` build options are deprecated.
Please use `addons.exclude` or `addons.include` respectively instead.

### Before

```js
// ember-cli-build.js

const EmberApp = require('ember-cli/lib/broccoli/ember-app');

module.exports = function (defaults) {
  const app = new EmberApp(defaults, {
    addons: {
      blacklist: ['ember-freestyle'],
    },
  });

  return app.toTree();
};
```

### After

```js
// ember-cli-build.js

const EmberApp = require('ember-cli/lib/broccoli/ember-app');

module.exports = function (defaults) {
  const app = new EmberApp(defaults, {
    addons: {
      exclude: ['ember-freestyle'],
    },
  });

  return app.toTree();
};
```

## How We Teach This

At the moment, these options are not mentioned in the Ember CLI guides.
This means, no documentation updates are required.

## Drawbacks

- I cannot see any real drawbacks at the moment, the only user action that is
required to resolve the deprecation is to rename one or two configuration keys

## Alternatives

- I cannot think of any alternatives worth mentioning, keeping both sets of keys
doesn't seem like a good idea, because they provide exactly the same functionality

## Unresolved questions

- None at the moment
