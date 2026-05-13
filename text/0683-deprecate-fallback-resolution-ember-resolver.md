---
Start Date: 2020-11-23
Relevant Team(s): Ember.js, Learning
RFC PR: https://github.com/emberjs/rfcs/pull/683
Tracking: (leave this empty)
---

# Deprecate Fallback Lookup Paths in ember-resolver

## Summary

Currently we have multiple fallback lookup paths in ember-resolver. As we continue to migrate towards a more strict future where imports are verified at build time (https://github.com/emberjs/rfcs/pull/454) it will be helpful to remove non-strict lookups from ember-resolver, which support many-to-one lookups, in favor of a one-to-one mechanism. This will enable us to make codemods to migrate to:
* Strict Mode (https://github.com/emberjs/rfcs/pull/496)
* the v2 addon format (https://github.com/emberjs/rfcs/pull/507)
* the slimming down the ember-cli build pipeline (https://github.com/emberjs/rfcs/pull/578)
* strict service lookups (https://github.com/emberjs/rfcs/pull/585)

## Motivation

As the community is moving forward with strict lookups having this done in ember-resolver will allow developers to slowly migrate libraries and applications to take advantage of imports verified at build time, the v2 addon format, Strict Mode and Strict Service Lookups. Without ember-resolver support to ensure that applications and libraries use strict lookups, migrating the ecosystem will require application developers using new features to discover issues when integration occurs.

Switching to the strict resolver by default and deprecating the classic resolver opens a migration path for applications that want to ensure compatibility with the new v2 package format. Additionally, apps using the strict resolver will see significant boot-up performance wins, because lookup costs become O(1) instead of O(n).

Ember apps and addons currently default to using https://github.com/ember-cli/ember-resolver to lookup values from the registry. These values include:

* Services
* Components
* Helpers
* Modifiers
* Templates

ember-strict-resolver defines a stricter and therefore much faster version of these lookups, which results in a 1:1 instead of 1:N resolution.

In the flagship web app at LinkedIn, adopting ember-strict-resolver resulted in the following improvements (per a [TracerBench](https://www.tracerbench.com/) analysis):

* `boot` phase estimated difference -188ms [-195ms to -181ms]
* `transition` phase estimated difference -42ms [-57ms to -27ms]
* `render` phase estimated difference -23ms [-37ms to -9ms]

## Detailed design

As libraries migrate to the new use strict format older applications will not be negatively affected as the strict subset resolution is backwards compatible from libraries to applications.

The major difference between the two resolvers can be easily linted against:

_It is important to call out that app re-exports are necessary when referencing components coming from an external source or using Wall Street Syntax._

| Lookup                                                  | Ember-resolver                                                                                                                                                            | ember-strict-resolver                                    |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| service/foo‌ ‌                                          | service:foo‌ ‌                                                                                                                                                            | service:foo‌ ‌                                           |
| service/foo-bar‌ ‌                                      | service:foo-bar,‌ ‌service:fooBar‌ ‌                                                                                                                                      | service:foo-bar‌ ‌                                       |
| (external-module-name)/addo‌n/services/foo-bar‌ ‌       | service:(external-module-nam‌e)@foo-bar‌ ‌service:(external-module-name)@fooBar‌                                                                                          | service:(external-module-nam‌e)@foo-bar‌ ‌               |
| controller/foo/bar‌ ‌                                   | this.controllerFor(‘foo.bar’),‌ ‌this.controllerFor(‘foo/bar’)‌ ‌                                                                                                         | this.controllerFor(‘foo.bar’)‌ ‌                         |
| component/foo/bar‌ ‌                                    | ‌{{foo/bar}}‌ ‌                                                                                                                                                           | {{foo/bar}}‌ ‌                                           |
| component/foo/bar-baz‌ ‌                                | ‌{{foo/bar-baz}},‌ ‌{{foo/bar_baz}}‌ ‌                                                                                                                                    | ‌{{foo/bar-baz}}‌ ‌                                      |
| (external-module-name)/addo‌n/components/foo/bar-baz‌ ‌ | {{component‌ ‌“(external-module-name)/foo/‌bar-baz”}}‌ ‌{{component‌ ‌“(external-module-name)/foo/‌bar_baz”}}‌ ‌{{component‌ ‌“(external-module-name)/foo/‌barBaz”}}‌ ‌ ‌ | {{component‌ ‌“(external-module-name)/foo/‌bar-baz”}}‌ ‌ |

### Timeline

Make the changes to turn on the strict mode lookups in ember-resolver under a flag

As mentioned above turning on the strict mode should be added as a patch update to ember-resolver with a deprecation message that it will be turned on as a default operation in the next major version. This option would be set in the `ember-cli-build.js` for ember-resolver as the following:

> A reason for making this option as a part of ember-cli-build.js will allow the build system to only bundle the strict resolver. This could be argued as a limitation for toggling between a strict and non-strict mode.

```
   "ember-resolver": {
     strict: true,
   }
```

Make the strict resolver the default for new applications and addons.

Making the resolver default to be turned on in strict mode in the default ember addon or application template is important as new addons being created should have an eye towards compatibility with the new system.

Document how to migrate to strict resolution.

This will have to be done in ember-resolver and in the ember guides to ensure that all new addons have the correct information when adding new content to the ecosystem and for those migrating. Lint rules can be added to eslint-plugin-ember as well as ember-template-lint to ensure that addons and applications that are interested in migrating are able to gauge the requirements. There will need to be additional tooling added in ember-cli or in ember-resolver to understand if any downstream dependencies of an application or addon do not support strict mode. As stated above leaf nodes migrating to strict mode do not impact upstream dependencies ability to consume them but if upstream dependencies update before downstream dependencies there is a possibility for runtime issues (these will be caught in tests as lookups will just fail to resolve).

Deprecate the loose-mode resolver.

Deprecating the loose-model in ember-resolver should happen in the next major version to avoid large ecosystem repercussions due to capability issues.

### Questions

* can an app turn on the strict resolver before all of its addons have migrated?
* can addons switch to using the strict resolver internally and still be consumed by an app using the classic resolver?

## How we teach this

_What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing Ember patterns, or as a
wholly new one?_

This change actually reduces the friction for both new and existing users, since currently there are multiple ways to access the registry. Older codebases exhibit a mix of syntax and legacy lookup logic, which will be cleaned up by adopting the strict resolver.

_Would the acceptance of this proposal mean the Ember guides must be
re-organized or altered? Does it change how Ember is taught to new users_
at any level?

The current guides currently adhere to a strict lookup policy from my searching. To guarantee it does not regress, the Ember guides tutorials migrate to the strict flag in ember-resolver such as https://github.com/ember-learn/super-rentals/tree/super-rentals-tutorial-output.

_How should this feature be introduced and taught to existing Ember
users?_

The gradual migration would be an opt-in flag introduced in a minor version of https://github.com/ember-cli/ember-resolver and then turned on as the default in the major version after that.

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Ember,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.

A reason to not move the ecosystem forward in this regard is to prioritize technologies that will remove the need for the registry as a sub-system of Ember all-together. Technologies like template imports and reworking the service injection system to not require a resolution mechanic to function.

This migration would introduce a level of churn to the ecosystem as it requires all libraries and applications to become more strict with their resolution, but does benefit the tightening of the interface to interact with the registry for potentially more functionality in the future such as Strict Service Lookups.

This is an iterative performance win for the ecosystem. Future enhancements such as Template Imports will unlock additional wins on top of this but not gate boot performance benefits.

## Alternatives

> What other designs have been considered? What is the impact of not doing this?

Don't do this, and just make the jump to full strict mode to unlock performance wins. This would mean that all performance improvements are gated by the use of strict-mode templates, rather than something that can be done incrementally. Not reducing the surface area for dynamic lookups will force the ecosystem to adopt template imports for all their dependencies to unlock performance benefits.

> This section could also include prior art, that is, how other frameworks in the same domain have solved this problem.

Major frameworks (React, Vue, Svelte, etc) ecosystems do not do dynamic lookup the way Ember does, but implore the use of node resolution to get templates or components. Ember’s path towards this solution comes with the use of template imports and the Embroider packaging v2 spec. Reducing the many to one lookup and eventually migrating from the dynamic lookup cases for components and templates to use node resolution via template imports and the v2 packaging spec will allow the ecosystem to gradually migrate.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
> TBD?

What is the migration path and support for pods?

## Appendix

This document was written based on debugging how often we call into the resolver to access values from the registry, on an initial page load of our homepage we access the registry ~11,800 times.
We utilized initial findings from running the no-implicit-this lint rule to estimate our migration effort, and then ran the no-implicit-this codemod gradually across the codebase. As a side benefit, this helped us in running the angle-brackets codemod .
A footgun we found when running the migration was ensuring that our data-test-selector instances used in curly components to be implicit booleans which we found by helping contribute https://github.com/ember-template-lint/ember-template-lint/blob/master/docs/rule/no-positional-data-test-selectors.md and running the fixer across our codebase.
