- Start Date: 2019-12-17
- Relevant Team(s): Ember.js, Ember Data, Ember-CLI
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Deprecation Shaking

## Summary

Allow applications and addons to eagerly opt-in to "shaking" deprecated
features, removing them from the builds of apps that do not use them.

## Motivation

Ember has always had strong guarantees around semver compliance and a solid,
stable upgrade path for every major change it makes. This has, over time, led
to a large amount of extra code that exists in the framework - code that is
marked for removal in the future, but can't be removed immediately due to those
guarantees.

This is especially true now that with Octane, which includes a large new set of
features that were ultimately designed to completely replace lots of older
features. Classic features aren't deprecated yet, but at some point in the
future they likely will be, and being able to take advantage of those size
reductions as they come would be a huge selling point for Ember apps.

## Detailed design

This RFC proposes allowing users to specify deprecation compliance for both apps
and addons via a field in the `ember` property in `package.json`:

```json
{
  "ember": {
    "edition": "octane",
    "deprecationCompliant": {
      "ember-source": "3.15.0"
    }
  }
}
```

Deprecation compliance can be specified for any Ember addon, including
`ember-source`. Specifying compliance for a particular version of an addon means
that the consuming app/addon will function without triggering any deprecations
that were added in or prior to that version. Specifying the version number will
turn all deprecations before the version into _assertions_ when running in DEBUG
mode, preventing users from relying on the behavior altogether, but still
providing a helpful error message in those cases. In production builds of
`ember-source`, the features will be removed as possible, but they are not
guaranteed to be removed fully.

The `deprecationCompliant` field must be an exact, existing version of the
addon. New applications and addons will have `deprecationCompliant` set to the
latest version of `ember-source` and `ember-data` once the feature has landed.

Ember's `deprecate` method will receive two new arguments that allow it to
participate in this system:

```ts
interface DeprecationOptions {
  id: string;
  until: string;
  since: string;
  source: string;
  url?: string;
}

declare function deprecate(message: string, test?: boolean, options?: DeprecationOptions) => void;
```

`since` must be a SemVer version equal to or lower than the current version of
the library. `source` must be the name of the library that is creating the
deprecation. If either of these options is missing, then the deprecation will
not be affected by any settings for deprecation compliance in consuming
applications. Not including these options will be deprecated and issue a
warning.

### Dependencies

Apps and addons may depend on other addons that are not yet deprecation
compliant with a given version. If the parent app or addon has a more recent
version of a library specified, an error will be thrown, alerting the user of
the addon that is not compliant. The user can then lower their deprecation
compliance version to match the addon if they choose, but they _cannot_ override
the compliance versions of their dependencies, since there is no way for us to
know whether or not the feature is used and will cause errors otherwise.

If an addon does not specify a deprecation compliance version, then it will be
assumed to be compliant.

### A Note on Per-Feature Shaking and Optional Features

An [earlier version of this feature](https://github.com/emberjs/rfcs/issues/532)
proposed allowing users to specify which features they wanted to disable
explicitly. The thinking was that this would prevent deprecations from being
blocked on each other, and enable more aggressive deprecation timelines in
general.

However, attempting to do this in a generic way would have resulted in a
combinatorial explosion of possible deprecations that could interact with one
another. If a user chose, for instance, to enable shaking for observers, but not
for computed properties, we may not actually be able to know if we could safely
shake all of the related code, since they share a lot of the same
infrastructure. It would also be very difficult to _test_ all possible
combinations of enabled/disabled deprecations. Shaking all deprecations prior to
a minimum version gives us one target combination, which is much easier to
reason about internally, and much easier to test.

Instead, we can use Ember's existing **optional features** infrastructure to
allow users to disable larger, more embedded features within Ember eagerly.
Optional features are generally reserved for larger shifts in Ember's behavior,
so there are fewer combinations to test compared to deprecations, which are used
for large and small things.

Optional features also send a better signal to the community. Major features
like Classic classes, observers, and computed properties can first go through a
period of being optional, before eventually being deprecated, and then removed.
Transition to an optional feature is a signal that the feature will eventually
be removed, but not in the near future, so it's not an
upgrade-now-or-forever-be-stuck-on-the-last-major-version-of-Ember level
priority. The feature can be converted to a true deprecation when the community
is ready and most apps and addons have converted to new idioms.

## How we teach this

This should be included in a guides section on performance and build size. It's
an advanced topic, so it should not be placed in a section aimed toward
beginners.

### API Docs

#### `deprecationCompliant`

This option specifies the versions of libraries that the application or addon is
_deprecation compliant_ with. Deprecation compliance means that the app/addon
does not use any features that have been deprecated on-or-before that version,
and does not trigger any warning messages when run.

```json
{
  "ember": {
    "edition": "octane",
    "deprecationCompliant": {
      "ember-source": "3.15.0"
    }
  }
}
```

Telling Ember the version that the app/addon is compliant with will cause all
deprecations for that library prior to the specified version to throw errors
instead of warnings in development builds. Libraries may also attempt to remove
that code whenever possible in production builds, slimming down the final build
and app size.

`deprecationCompliant` should be set to an object whose keys are the names of
packages that the app/addon depends on, and whose values are the exact SemVer
version that the app/addon is compliant with.

##### Dependencies

The deprecation compliant version of an app/addon cannot be greater than any of
their dependencies compliant versions. Ember CLI will throw an error if it finds
that one of your dependencies has an older version, and will let you know what
version you can lower your own compliance to.

If a child addon does not specify compliance for a given library, it is assumed
to be compliant, and will not cause errors at build time. The library may still
not be compliant however, and could cause assertions to be thrown, so you should
be aware of this and report the issue to the library directly if this occurs.

## Drawbacks

- This increases complexity for addon authors, who will likely receive pressure
  to update their compliance more frequently from apps that make use of these
  features.

## Alternatives

- A "flag" version of deprecation shaking, where users decide which features to
  disable and attempt to shake. This has several problems, which are covered
  above in the Detailed Design section.

## Unresolved questions

- Should `ember-cli-update` attempt to automatically update the compliant
  version?
- Should Ember provide any utilities for checking and shaking deprecated
  features at build time for other addons?
