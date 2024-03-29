---
stage: recommended
start-date: 2017-02-02T00:00:00.000Z
release-date: 2017-04-29T00:00:00.000Z
release-versions:
  ember-cli: v2.13.0

teams:
  - cli
prs:
  accepted: https://github.com/ember-cli/rfcs/pull/96
project-link:
---

# Summary

Enable Ember CLI users to opt into using yarn for packagement management.

# Motivation

Ember CLI currently uses the npm command line tool to install dependencies when you run `ember install` or `ember new`/`ember addon`. However, several problems usually arise from npm's semantics.
Dependency resolution and install times can be significant enough to disrupt workflows, as well as offline support, non-deterministic, non-flat dependency graphs.

Yarn was introduced to the JavaScript community with the intent to provide a better developer experience in these areas:
- Faster installs
- Offline support
- Deterministic dependency graphs
- Lockfile semantics

While Ember CLI users can currently use Yarn to manage their dependencies, Ember CLI will use the npm client internally when running the above mentioned commands. By allowing users to specify that Ember CLI should use Yarn for everything, we're hoping to provide a more consistent experience.

# Detailed design

Enabling Yarn is designed as opt-in to prevent disruptions to the developer's current workflow.
We will address the two moments where this can happen.

## `ember install`

There are two mechanisms through which to opt-in.
The first one is the presence of a `yarn.lock` file in the project root.

The `yarn.lock` file is generated by Yarn when you run `yarn install` (or the shorter `yarn`),
so we assume that its presence means the developer intends to use Yarn to manage their dependencies.

Alternatively you, you can force Ember CLI to use Yarn with the `--yarn` flag, and symmetrically,
you can force Ember CLI to not use Yarn with the `--no-yarn` flag.

To recap:

- `ember install ember-cli-mirage` with `yarn.lock` present will use Yarn
- `ember install ember-cli-mirage` without `yarn.lock` present will use npm
- `ember install ember-cli-mirage --yarn` will use Yarn
- `ember install ember-cli-mirage --no-yarn` will use npm

## `ember init`, `ember new`, `ember addon`

Since this triad of commands is generally ran before a project is set up, there is no `yarn.lock` file presence to check.
This means we are left with the `--yarn`/`--no-yarn` pair of flags, that will also be added to these commands:

- `ember new my-app` will use npm
- `ember new my-app --yarn` will use Yarn

The above also applies to `ember addon` and `ember init`, noting that `ember init` doesn't receive any arguments.

# How We Teach This

Both the Ember.js Guides as well as the Ember CLI Guides will be updated to reflect the new flags,
as well as the new semantics of `ember install` in the presence of `yarn.lock`.

In addition, the built-in instructions for `ember help` will be updated to reflect this.

# Drawbacks

To be determined.

# Alternatives

Do nothing.

# Unresolved questions

To be determined.
