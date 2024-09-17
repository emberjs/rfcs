- Start Date: 2018-12-24
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Build in ember-exam

## Summary

This RFC proposes to promote [Ember-exam](https://github.com/ember-cli/ember-exam)'s functionality
to be a core part of the developer workflow.

## Motivation

Ember cares greatly about compatibility. Every addon is generated with a config file for `ember-try`
to ensure the code is tested in the last two LTS version of Ember and on the upcoming _beta_ and _canary_
channels.

We are doing great but we can still do better.

Fast test suites are critical to the adoption of good testing practices.
Testing in several browsers is, specially in addons, is key to ensure your code will run correctly
for all your users and to keep the web open.

Although testing in multiple browsers is already possible today out of the box,
going from testing on only one browser to testing two or three doubles or triples the testing time
and it is a burden many people do not care to face.

[Ember-exam](https://github.com/ember-cli/ember-exam) is an fantastic addon that allows users to
run their tests in parallel in different browsers or even split their suite in chunks and run it
in several instances of the same browser. It allows users to run their tests in a fraction of the time
or increase the number of tested browsers without worsening the feedback loop.

It is just too good for an opinionated framework like Ember.js to not advocate its usage.

## Detailed design

There are two different approaches for this goal, listed in order of complexity:

### Add ember-exam to the default blueprint for apps and addons

This approach yields a good return with a relative low effort. Including it by default on newly generated
projects signals it to the community as a good practice. Ember-exam is already part of the ember-cli
organization so there shouldn't be a problem controlling the code and the pace of the releases.

Developers can totally opt out by just removing the addon should they want to.

The downside of this approach is that we cannot just assume that everybody will have ember-exam-like features.
If we wanted to implement some advanced features in ember-cli like testing tailor-made per-browser builds, we'd
have to account for that fragmentation.

### Merge ember-exam's functionality into the testing experience.

A more involved approach consists on moving the code that powers ember-exam builds into the relevant
libraries (ember-cli / ember-test-loader / ember-qunit / ember-cli-mocha) and discontinuing Ember-exam as a separate tool.

On one hand side ember-exam wouldn't have to monkey patch the test-loader anymore and Ember would
completely embrace parallel testing, while on the other hand the complexity needed for supporting
parallel testing might end divided across several projects.

### Default configuration

In both scenarios we can start with a parallel count of 1, meaning parallel builds are still opt-in.
With this default configuration there won't be any discernible difference from how things work today out of the box.

User will only have to make a change in the `/testem.js` config file to start getting the advantages of parallel testing.

## How we teach this

The testing section of the guides will have to be amended to include information about splitting, parallelization
and other concepts that are today explained in the [Readme of Ember-exam](https://github.com/ember-cli/ember-exam/blob/master/README.md)
test suites

## Drawbacks

Adding new features to the default Ember experience means more code to maintain and more
features for newcomers to learn, who already have plenty on their plate.

We can limit the teaching impact of parallel builds by defaulting to 1, but does not remove it completely.

## Alternatives

Leave ember-exam as the nice addon it is today so users who care the most for speed or multi-browser
support install it.

## Unresolved questions

By default Ember addons are only tested in headless chrome. Although it's not _directly_ related parallel
testing it could make multi-browser testing cheap enough to change the defaults so addons are tested in both
chrome and firefox, ensuring better compatibility.