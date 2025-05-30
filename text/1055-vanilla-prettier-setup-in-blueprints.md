---
stage: recommended
start-date: 2024-12-03T00:00:00.000Z
release-date:
release-versions:
  ember-cli: 6.3.0
teams:
  - cli
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1055'
  ready-for-release: 'https://github.com/emberjs/rfcs/pull/1063'
  released: 'https://github.com/emberjs/rfcs/pull/1073'
  recommended: 'https://github.com/emberjs/rfcs/pull/1097'
project-link:
suite:
---

# Vanilla Prettier Setup in Blueprints

## Summary

This RFC proposes to migrate to a vanilla Prettier setup in the blueprints, instead of running Prettier via ESLint and Stylelint.

## Motivation

1. Because we run Prettier via ESLint and Stylelint, we only run the files these linters support through Prettier - Using a vanilla Prettier setup, would format all files Prettier supports, ensuring even more consistency in projects
2. Less dependencies in the blueprints - `eslint-plugin-prettier` and `stylelint-prettier` would not be needed anymore
3. The Prettier team recommends running Prettier directly, and not via linters:
   - Running Prettier directly is faster than running Prettier via ESLint and Stylelint
   - ESLint and Stylelint show red squiggly lines in editors (when using the corresponding extensions), while the idea behind Prettier is to make developers forget about formatting

`3.` is mostly taken from [Integrating with Linters > Notes](https://prettier.io/docs/en/integrating-with-linters.html#notes)

## Detailed Design

We would add the following scripts to the `package.json` file in the `app` blueprint:

```diff
+ "format": "prettier . --cache --write",
+ "lint:format": "prettier . --cache --check",
```

- `lint:format` would check the formatting of _all_ files Prettier supports
- `lint:format` would also run when running the `lint` script
- `format` would format _all_ files Prettier supports
- `format` would also run when running the `lint:fix` script

> NOTE: We use `format` instead of `lint:format:fix`, because we don't want to run Prettier parallel to ESLint and Stylelint when fixing lint errors. The `lint:fix` script will be updated to always run `format` last.

We would remove the following dependencies from the `package.json` file in the `app` blueprint:

```diff
- "eslint-plugin-prettier": "*",
- "stylelint-prettier": "*",
```

As these would not be needed anymore.

> NOTE: We will keep `eslint-config-prettier` installed, as we need this package to turn off the stylistic ESLint rules that might conflict with Prettier.

We would update the `.prettierignore` file in the `app` blueprint:

```diff
+ /pnpm-lock.yaml
```

To make sure Prettier does not format pnpm's lockfile.

We would also need to make sure that every file generated by the `app` blueprint is formatted correctly.

## How We Teach This

N/A

## Drawbacks

- Some developers or teams prefer running Prettier via ESLint and Stylelint

## Alternatives

N/A

## Unresolved Questions

N/A
