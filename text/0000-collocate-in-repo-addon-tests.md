- Start Date: 2018-09-05
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Collocate in-repo-addon tests

## Summary

Locate tests that are associated with an in-repo-addon in that addon's tests folder. However, for this RFC, the output of the tests bundles will remain identical to the current output.

## Motivation

Tests that are collocated next to their code in addons will be more discoverable and maintainable.

Having a simple syntax for executing tests associated with an addon is useful and time-saving.

## Detailed design

- Indicate that an addon has tests that are collocated
  - use `includeTestsInHost: true` in `index.js` to indicate collocated tests
- Implement CLI flag that only runs the tests for a specified addon
  - `ember test --ir-addon foo-bar`
  - The flag's use would create a list of tests to run that are collocated in that in-repo-addon's test folder. The tests will still be included all in one bundle, so this is basically just running the modules that are associated with the co-located tests.

## Not covered in this RFC (but will be covered in a following RFC)
- Splitting the test bundles
  - Tests should be separately bundled to allow running these tests without loading other tests.
  - In very large application with many in-repo-addons, there is significant overhead in loading all of the application's tests, when only a single in-repo-addon's tests are being tested
  - Expect another RFC coming soon to address this functionality
- Option to run addpn tests with dummy app
  - Tests for an in-repo-addon/engine could be chosen to either run against the main app, or a (shared) dummy app.
  
## How we teach this

- Create a guide chapter on in-repo-addons
  - Add information about this feature to that chapter

## Drawbacks

As the use of this feature would be completely optional by a user any drawbacks could be mitigated by not opting in.

## Alternatives

- Use isDevelopingAddon to signal test inclusion

## Unresolved questions


