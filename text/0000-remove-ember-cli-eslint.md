- Start Date: 2018-08-13
- RFC PR: (leave this empty)

# Summary

Remove https://github.com/ember-cli/ember-cli-eslint from projects `ember-cli` generates.

# Motivation

1. Improve our build speed
2. Hacks required to support features such as [PR #122
   broccoli-lint-eslint](https://github.com/ember-cli/broccoli-lint-eslint/pull/122#discussion-diff-153937455R28)
3. Simplicity. `eslint` is common among JS stack, and integrations with editors
   / precommit-hooks are ubiquitous. Removing this layer of abstraction will
   simplify how `eslint` is used throughout `ember-cli`.

# Detailed design

1. Change blueprint to pull in `eslint` as opposed to `ember-cli-eslint` under
   `devDependencies`.
2. Provide documentation on `eslint` and editor integration as well as precommit hooks


# How We Teach This

Providing documentation should suffice. Also, going towards a much more explicit
path, this should become _easier_ to teach because of less abstractions.

# Drawbacks

1. No console warnings during builds
2. lint failures are no longer included in browser tests

# Alternatives

N/A

# Unresolved questions

N/A
