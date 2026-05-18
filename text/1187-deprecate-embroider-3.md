---
stage: accepted
start-date: 2026-05-17T00:00:00.000Z
release-date:
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1187
project-link:
---

# Deprecate Embroider@3

## Summary

Embroider+Vite is now the default build system for any newly generated Ember application. To achieve this implementation we needed to release a breaking change of Embroider (Embroider@4) which represents a complete re-architecture of how the Embroider build system works. 

As Embroider@4 was a breaking change, that required some manual migration steps to upgrade to, the Ember Core Tooling Team has been maintaining the Embroider@3 branch to give developers time to upgrade/migrate.

This RFC officially deprecates Embroider@3 and is a clear communication that no more maintenance will be done on the old version. All apps should upgrade to Embroider@4 as soon as possible.

## Motivation

The first version of Embroider that was in any official default blueprint was Embroider@4, which was included in [Ember@6.8 when Embroider+Vite became the default build system](https://blog.emberjs.com/ember-released-6-8#toc_embroider-and-vite-by-default). There have been other application blueprints available for early-adopters for quite some time, but the Ember Core Tooling team was never confident enough in the implementation to make it the default for everyone. This was not a negative view of the Embroider code itself, it was more a lack of confidence in one of the core architectural decisions of the old approach. Embroider@3 wrapped Webpack inside the existing ember-cli broccoli-based build system and was a fundamental limiting factor that prevented Ember from taking advantage of improvements in the modern build tooling ecosystem.

Embroider@4 was a significant refactoring of the architecture and approach of building Ember apps with modern tooling (such as Vite), where Embroider no longer wrapped the bundler in an ember-cli pipeline but instead Embroider acted as a **plugin** for the bundler. This meant that we could make use of the improvements from the respective bundler's dev servers without having to implement all of those features as a pass-through for ember-cli.

The Ember Tooling Team decided to keep maintaining the legacy Embroider@3 branch (with bugfixes and security patches) so that we would give early adopters who had already opted-in to Embroider@3 time to migrate to the new system.

We knew that we couldn't maintain the legacy Embroider@3 branch forever but we have recently discovered that because of the changes needed for the [using-amd-bundles deprecation](https://deprecations.emberjs.com/id/using-amd-bundles) it is practically impossible to support Ember@7 with Embroider@3 without any breaking changes. This fact has been the motivating factor for deprecating Embroider@3 now.

## Technical details

To be clear about the changes that will actually happen if this RFC is accepted I want to list out the exact changes that will happen during the implementation of the RFC:

- A new warning will be added to Embroider@3 in a patch release that communicates that Embroider@3 is deprecated and will link to documentation on how to upgrade
- The `stable` branch on the `embroider-build/embroider` repo will be renamed to `legacy` to confuse anyone

## Transition Path

### Applications

The transition path for Applications is the same as it was for anyone wanting to move from ember-cli to vite, the Ember Tooling Team recommends that you run the [ember-vite-codemod](https://github.com/mainmatter/ember-vite-codemod). This has well-tested support for upgrading applications from either classic ember-cli build pipelines or Embroider@3 build piplelines.

For a slightly easier migration experience we recommend that you migrate all of your templates to GJS/GTS using the [@embroider/template-tag-codemod](https://github.com/embroider-build/embroider/tree/main/packages/template-tag-codemod) before running the `ember-vite-codemod`. This will allow you to solve any issues that your own app might have with Vite on a file-by-file basis before moving the whole app.


### Addons

The current default addon blueprint has pre-configured `ember-try` scenarios provided by `@embroider/test-setup` for `embroider-safe` and `embroider-optimised` scenarios. It will not be possible to clear the deprecation for these scenarios, but instead the addon should migrate to the [new v2 addon blueprint](https://rfcs.emberjs.com/id/0985-v2-addon-by-default).


## How We Teach This

As Embroider@3 was never the default experience for anyone (and the main way to opt-into the Embroider@3 build system has [already been deprecated and removed in ember-cli@7](https://deprecations.emberjs.com/id/dont-use-embroider-option)) we should not need to upgrade any documentation.

For early adopters who have already opted-in to Embroider@3, we need to make sure that the deprecation notification is "loud" enough that it can't be missed and that it points to comprehensive documentation on how to upgrade.

## Drawbacks

### Not possible to clear deprecation in v1 addon blueprint

At the time of writing of this RFC the new v2 addon by default RFC has not been implemented, so it is not possible to clear this deprecation from your builds. Addons will also have to limit their embroider tests to a maximum Ember version of 6.12 since the `@embroider/test-setup` will never work with Ember@7.

## Alternatives



## Unresolved questions


