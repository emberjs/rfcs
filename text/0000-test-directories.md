- Start Date: 2020-01=09
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/575
- Tracking:

# Test Directories

## Summary

This RFC proposes that we change the names of an Ember app's test directories to reflect the names
in the guides. For example, instead of `tests/acceptance/*-test.js`, we should have
`tests/application/*-test.js`.

## Motivation

Renaming test directories to match the Guides will make it easier to have a shared language
between tests, and to reduce confusion for newer users when using Ember CLI.

## Detailed design

These are the directory transformations that need to happen:

- tests/acceptance -> tests/application
- tests/integration > tests/rendering
- tests/unit/utils -> tests/unit
- tests/unit/controllers -> tests/container/controllers
- tests/unit/routes -> tests/container/routes
- tests/unit/services -> tests/container/services

The changes required are:

- Updating the default blueprint for Ember apps
- Updating the generators for these types of tests to create files in the right directories with the right module names.
- Providing a codemod or script to move these files around.

## How we teach this

This new lexicon is already in the Ember guides, but it's possibly that a blog post noting the old
verbiage would be useful as a point of reference.

## Drawbacks

Almost no code will change (except the string names of test modules), so it's possible that this
will be seen as unnecessary churn.

## Alternatives

An alternative would be do document that when the guides talk about Application tests, they are
referring the `tests/acceptance` directory, and so on.

## Unresolved questions

- There's a possibility that there are Ember apps out there that are not using the default directory
structure provided by Ember CLI. If a custom directory structure is used, what is the way to solve this?
- Apps using `pods`. I'm not sure of the directory structure for tests created by pods. It's not
clear if this RFC should include trying to codemod those.
