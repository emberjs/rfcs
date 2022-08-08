---
stage: accepted
start-date: 2020-01-14T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/704
project-link:
---

# Deprecate Octane Optional Features

## Summary

Deprecate the optional features introduced in the transition to Ember Octane.
This specifically refers to the optional features which are required for an
Ember application to set the `octane` edition:

- `application-template-wrapper`
- `template-only-glimmer-components`

## Motivation

The Octane edition consisted for the most part of backwards compatible changes,
a design choice inline with the Ember philosophy of introducing new features and
changes while ensuring that there is an upgrade path. Editions are about
introducing a new programming model side-by-side with the old one, allowing
users to adopt the changes gradually. As such, enabling Octane is almost
identical to any other Ember upgrade, with one exception - required optional
features.

While most changes in the Octane programming model were backwards compatible,
there were a few small tweaks to behaviors that could not be done in a backwards
compatible way. In order to fully enable Octane, users had to both upgrade to
the minimum version of Ember, _and_ toggle these behaviors via optional
features.

The two optional features that were required for Octane are:

- `application-template-wrapper`, which must be set to `false` in Octane
- `template-only-glimmer-components`, which must be set to `true` in Octane

While there are other optional features available which Ember users should
adopt, they are not requirements for Octane specifically.

As part of the post-Octane cleanup, this RFC proposes deprecating these optional
features, so that the Octane behavior is the only supported behavior. This will
simplify user configuration, and enable cleanup of the code that supports these
optional features internally.

## Transition Path

This deprecation will be a 2 phase deprecation:

1. Deprecate the following optional feature flag settings:
  - `application-template-wrapper: true`
  - `template-only-glimmer-components: false`
2. Deprecate specifying the optional feature at all

### Phase 1

The first phase will require that all users have toggled the optional feature to
the correct position for Octane. This phase can be implemented immediately, and
will position us to be able to enter phase 2 in the next major version after
implementation (currently v4).

Since the specifying the optional feature will cause it to be set to the
incorrect value in Ember v3, users will have to specify the optional feature
explicitly.

### Phase 2

Once a major version has been released after phase 1 has been implemented, all
users will be using the same setting for these optional features, explicitly. In
the first release of Ember v4, we will be able to safely change the default
value of the optional feature flags to be the correct Octane setting. Users will
be able to drop the explicit setting at this point, and the blueprint will be
updated to drop it as well.

At this point, the default setting will be the Octane-compatible one, and
explicitly setting it to the Octane-compatible one will be allowed. Explicitly
setting it to the non-Octane setting will trigger an assertion. This means that
users can only possibly have one behavior specified, and we can deprecate
specifying the optional feature at all.

## How We Teach This

### Deprecation Guide

#### `application-template-wrapper`

Setting the `application-template-wrapper` optional feature to `true` has been
deprecated. You must set this feature to `false`, disabling the application
wrapper. For more details on this optional feature, including the changes in
behavior disabling it causes and how you can disable it, see the
[optional features section](https://guides.emberjs.com/release/configuring-ember/optional-features/#toc_application-template-wrapper)
of the Ember guides. You can also run `npx @ember/octanify` to set this feature
to the correct value.

#### `template-only-glimmer-components`

Setting the `template-only-glimmer-components` optional feature to `false` has been
deprecated. You must set this feature to `true`, enabling the template-only
Glimmer components. For more details on this optional feature, including the
changes in behavior enabling it causes and how you can enable it, see the
[optional features section](https://guides.emberjs.com/release/configuring-ember/optional-features/#toc_template-only-glimmer-components)
of the Ember guides. You can also run `npx @ember/octanify` to set this feature
to the correct value.

## Drawbacks

- Could cause churn in some existing applications

## Alternatives

- We could deprecate the other optional feature flags as well. While it is
  encouraged to toggle all optional feature flags, and all optional feature
  flags are toggled by default in new Ember apps, the other flags were required
  as part of Ember Octane and are conceptually separate. As such, they should be
  deprecated in a separate RFC.

