- Start Date: 2017-07-12
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

The release channel names in the Ember core projects (Ember, Ember Data, and
Ember CLI) today are: `release`, `beta` and `canary`. This RFC proposes
to change them to: `stable`, `beta` and `master`.

# Motivation

Why are we doing this? What use cases does it support? What is the expected
outcome?

The `release` release channel name is confusing since it literally applies
to *all* releases, not just the stable ones.

The `canary` release channel name seems alright in general, but is not
matching the branch name in the git repository (`master`) which makes
`npm install github:emberjs/ember.js#canary` fail.

This RFC aims to make the release channel names more intuitive and while
matching the git branch names.

# Detailed design

The proposal in this RFC is to rename `release` to `stable` and `canary` to
`master`.

Things that need to be updated:

- "Builds" webpage
- Release scripts in the repositories
- The `release` branches need to be renamed
- `ember-try` might need to be adjusted

# How We Teach This

It should be sufficient to explain the change in a blog post on emberjs.com.

# Drawbacks

The tradeoff is that references to `#release` and `#canary` will stop working
unless we publish to both old and new channel names for a while.

# Alternatives

The obvious alternative is keeping the current release channel names.

Alternative branch/release channel names were also considered
(e.g. `latest` which is the default in npm).

# Unresolved questions

- what exactly do we have to change so that `ember-try` still works properly?
