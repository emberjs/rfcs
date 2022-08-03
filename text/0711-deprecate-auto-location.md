---
# FIXME: This may be a further stage
Stage: Accepted
Start Date: 2021-01-23
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/711
---

# Deprecate AutoLocation

## Summary

Deprecate Ember.AutoLocation and related APIs. In next major version,
make history the default location (instead of auto).

## Motivation

In practice, 'auto' will almost always resolve to the 'history' location.

By removing the 'auto' location and setting 'history' as the default we're removing
some complexity around router location, with no downside for users, since they
all get it resolved to history anyway.

## Background

The Ember Router can use different mechanisms to serialize its state.
The two common ones are 'history' and 'hash'. There is also a special
location called 'auto' which does feature detection, preferring 'history'.

When the special 'auto' location was added (and made default), less than
half of browsers supported the (then) new history location API. For seamless
upgrade to history API when available, Ember created the special auto location.

These days (many years later), basically all browsers will get 'history',
since the History Location API is very widely supported, even by IE 11:
https://caniuse.com/mdn-api_history_pushstate

The check 'auto' is basically doing this:

```js
// Boosted from Modernizr: https://github.com/Modernizr/Modernizr/blob/master/feature-detects/history.js
// The stock browser on Android 2.2 & 2.3, and 4.0.x returns positive on history support
// Unfortunately support is really buggy and there is no clean way to detect
// these bugs, so we fall back to a user agent sniff :(

// We only want Android 2 and 4.0, stock browser, and not Chrome which identifies
// itself as 'Mobile Safari' as well, nor Windows Phone.

if (
    (userAgent.indexOf('Android 2.') !== -1 || userAgent.indexOf('Android 4.0') !== -1) &&
    userAgent.indexOf('Mobile Safari') !== -1 &&
    userAgent.indexOf('Chrome') === -1 &&
    userAgent.indexOf('Windows Phone') === -1
  ) {
    return false;
  }

  return Boolean(window.history && 'pushState' in window.history);
```

## Transition Path

We'll begin with deprecating auto (explicitly or implicitly set).

In next major release we change the default to 'history'.
Since auto is default today, and almost always resolve to history,
most users can change to 'history' and it's done.

We also need to update the template in ember-cli blueprint,
so that 'history' will be the default value for `locationType`
in environment.js.

There are two groups to consider:

1. People who need auto-detection (hash/history switching) and
2. those with location set to 'auto' (this will be most people).

The first group can implement feature detection,
via e.g. Modernizr[1], like this:

```js
// app/router.js
export default class Router extends EmberRouter {
  location = (historyFeatureDetection() ? 'history' : 'hash');
  // …
}
```

For the second group, they need to replace their choice
of 'auto' for 'history' in `locationType` in environment.js (or
`location` in router.js), since history is what 'auto' mostly
resolves to anyway.

### What To Deprecate (and later remove)

- The AutoLocation class
- The method `detect` from the `Location` interface
- Explicitly setting 'auto' as the preferred location.

### Deprecated or already private (should also be removed)
- Remove the `create` method on Location (replaced by registry)
- `registerImplementation` method on `Location`
- `supportsHistory` function exported in location/utils.ts

## How We Teach This

- Remove the text about auto location from guides.
- There is already good documentation about the different
  location types (history, hash, none) and how to write your own.
- Slight adjustments needed for the guides, removing mentions of auto[2].

Overall there should be fairly little changes to how Ember is taught. Since
the vast majority of users will need to make zero code changes and will have
zero change in actual behaviour.

## Drawbacks

Anyone using an unsupported browser will need to do feature detection
in their own code base or opt everyone into using the hash location.

https://caniuse.com/mdn-api_history_pushstate

## Alternatives

Do nothing.

## Unresolved questions

There are currently no unresolved questions.

## References

[1] https://github.com/Modernizr/Modernizr/blob/master/feature-detects/history.js
[2] https://guides.emberjs.com/release/configuring-ember/specifying-url-type/
