---
stage: released
start-date: 2022-12-15T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - typescript
prs:
  accepted: 'https://github.com/emberjs/rfcs/pull/889'
project-link:
---

# Deprecate @ember/error

## Summary

The @ember/error package is just a re-export of the native Error and is therefore unnecessary.

## Motivation

Removal of unnecessary code keeps the codebase cleaner and simplifies developer burden.

## Transition Path

There is no use case for this anymore. A simple codemod can convert current uses to the native Error class.

## How We Teach This

Remove @ember/error from docs.

## Drawbacks

You'll have to run a codemod to resolve existing usage. However, this is trivial and will actually simplify user code.

## Alternatives

None.

## Unresolved questions

None.
