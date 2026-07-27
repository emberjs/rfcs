---
stage: ready-for-release
start-date: 2026-06-16T00:00:00.000Z
release-date:
release-versions:
teams:
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/1214'
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

<!-- Replace "RFC title" with the title of your RFC -->

# Deprecate ember-template-lint

## Summary

`ember-template-lint` was created to provide a linting system for Ember templates, but it was created at a time when tools like `eslint` were not extensible enough to support different file types other than JS. `eslint` now supports multiple languages, and it is possible to implement all the same `ember-template-lint` rules in an `eslint` plugin.

This RFC proposes deprecating the use of `ember-template-lint` and unifying the community on a single place for Ember-specific lint rules: `eslint-plugin-ember`.

## Motivation

Since `eslint@v9` started to implement [custom language plugins](https://github.com/eslint/rfcs/blob/main/designs/2022-languages/README.md) it has become possible to implement all the `ember-template-lint` rules in `eslint` directly, and now that we use [GJS by default](https://rfcs.emberjs.com/id/0779-first-class-component-templates) for new components and route templates there are some benefits to moving our implementations to an `eslint` plugin e.g. rules like [builtin-component-arguments](https://github.com/ember-template-lint/ember-template-lint/blob/main/docs/rule/builtin-component-arguments.md) makes valid suggestions for the Ember-provided `<Input>` component, but it will trigger on any component named `Input`, even if you have implemented a different component in your application that you have named `Input`.

After moving all the lint rules to the `eslint` plugin, it is possible to allow the rule to check the import path for the `<Input>` component and only trigger errors when it is the `<Input>` component provided by Ember.

It is also better for the community to have a single place where all relevant lint rules are developed, rather than splitting the community's focus and development time between two projects.

## Detailed design

As of [eslint-plugin-ember@v13.0.0](https://github.com/ember-cli/eslint-plugin-ember/releases/tag/v13.0.0) all the existing `ember-template-lint` rules have been reimplemented in the `eslint` plugin. This RFC covers the deprecation of `ember-template-lint` and how we can move the community to depend on `eslint-plugin-ember` exclusively.

### Enable template lint rules by default

Although all the `ember-template-lint` rules have been implemented in `eslint-plugin-ember` already, none of them have been added to the default recommended set of rules. This was to allow people to opt-in to the new system without getting two notifications for every template lint violation. As part of this RFC we will be enabling all the rules that were enabled by default in `ember-template-lint` in the default configuration for `ember-template-lint` so that there is a lint-parity for newly generated applications.

Note: the default ruleset for `ember-template-lint` included a number of A11y focused rules and those rules should remain in the new set of default rules from `eslint-plugin-ember`. The EmberJS project has a core commitment to A11y and that should not change with this deprecation of `ember-template-lint`.

### Implement a deprecation warning in ember-template-lint

Anyone running `ember-template-lint` should see a loud deprecation warning to show that the project has been deprecated, and they should migrate to the new `eslint` configuration. This deprecation warning should link directly to the migration guide. 

### Support for HBS files

`eslint-plugin-ember` already supports using the migrated template lint rules on `hbs` files. A minor change will be needed to the `eslint` config to enable `eslint` to parse HBS files. This change can be made directly to the blueprint and detailed in a migration guide.

## How we teach this

The key teaching task for this RFC is to implement a migration guide. This can either be on the `ember-template-lint` repo as a simple Markdown file, or it could be a Markdown file on `eslint-plugin-ember` targeted at people migrating from `ember-template-lint`.

## Drawbacks

The fact that all the `eslint-plugin-ember` migrated lint rules are working as expected, with the same test cases, means that this should be a seamless migration for most people. I don't see any drawbacks.

## Alternatives

### Increase maintenance on ember-template-lint and related projects

One of the key motivations for this RFC is the lack of maintenance work on ember-template-lint and related projects. One key example is the fact that the custom `lint-todo` system that `ember-template-lint` implements [has a number of unfixed bugs that have not been addressed](https://github.com/ember-template-lint/ember-template-lint/issues/3211). 

An alternative to deprecating `ember-template-lint` is to dedicate a significant effort to improving maintenance and fixing a lot of these issues. I think this effort would be mostly wasted because there are already very good alternatives to the `lint-todo` system e.g. `eslint` has built-in support for a similar functionality called [bulk suppressions](https://eslint.org/blog/2025/04/introducing-bulk-suppressions/) and [Lint to the Future](https://github.com/mansona/lint-to-the-future) is somewhat popular in the Ember community.

## Unresolved questions

None that I know of
