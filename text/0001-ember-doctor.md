- Start Date: 2015-01-10
- RFC PR: (leave this empty)
- ember-cli Issue: (leave this empty)

# Summary

We need a way to run diagnostics on Ember CLI based projects to let developers know about potential system level incompatibilities. Developers should also be able to get a bill of health for their project for things like outdated dependencies.  This bill of health should also be extensible.

# Motivation

The motivation behind this is 2 pronged:

1. Allows developers to submit system level information in pull requests, so that bugs can be filed and potentially replicated.
2. Gives developers the ability to know about the health of their project and to potentially help with stagnation.

# Detailed design

The design for this is rather simple. We would first introduce a command called `ember doctor` that would run some default checks. The default checks would do the following:

-Run `ember v --verbose` and complain loudly for incompatible versions
-Run `npm outdated --depth 0` to check on outdated modules
-Run `bower list` and display out of date bower components

These are what is considered default `checks`.

In your project developers can setup their own Doctor `checks` that get merged in with the default checks. To allow for this Ember CLI will have `ember generate doctor check:service-health`.

This command will generate the following directory structure in the root of the project:

```
doctor/
  checks/
    service-health.js
  index.js
```

When `ember doctor` is ran we simply will do a merge of the default checks and the ones provided by the application.  

There should also be a way of excluding checks to be ran. Developers should be able to simply pass flags for things they do not care to run e.g. `ember doctor --skip-npm-version`.

# Drawbacks

This adds "yet another thing" to the Ember CLI API surface. Doctor will be bound to a network connection such as checking outdated dependencies.

# Alternatives

There have been other other attempts to put checking for system level checking in various places. The BDFL's would like to consolidate this into an `ember doctor` command.

# Unresolved questions
