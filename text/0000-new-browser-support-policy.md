- Start Date: 2020-11-28
- Relevant Team(s): All
- RFC PR:
- Tracking: (leave this empty)

# New Browser Support Policy

## Summary

Establishes a new browser support policy for the next major release of Ember and
Ember Data.

## Motivation

With Microsoft's recent release of the new Chromium-based Edge browser, which
has a compatibility mode for Internet Explorer built in, many frameworks, tools,
libraries, and websites have begun finally dropping support for the aging
browser. In order to unlock the latest browser features and continue improving
the framework as a whole, Ember should also drop support in the next major
release.

In dropping support for Internet Explorer, we will need a new browser support
policy. Until now Internet Explorer has been the "lowest common denominator"
for all features. It is a very old browser that no longer releases new versions,
and was not "evergreen" (e.g. constantly updating) even when it was. As such,
Ember has been very stable from release to release, as supporting any version of
Internet Explorer meant we simply could not use any new browser features.

With modern evergreen browser release cycles, where browsers release regularly
and users are generally on one of the more recent versions of a browser, this
dynamic changes. We cannot set an explicit version to support anymore, because
it would be impractical - Chrome alone releases a new version every 6 weeks.
Setting an explicit lowest-supported-version would lock us into having to
support a huge number of older browsers, many of which are entirely unused.

Conversely, however, tracking the latest release of a browser may not be enough
stability for Ember apps. If Ember were to adopt a new browser feature
immediately after its release, even though many web users were still using a
previous version, it would cause many Ember apps to either break, or be unable
to update until their users had also updated to the latest browser. Even a
rolling `N - 1` or `N - 2` support policy, where the last 1 or 2 versions are
supported, may not be enough. Many users were stuck in the recent Edgium update
for some time, for instance.

This RFC seeks to establish a new support policy that addresses these issues
with a new set of heuristics. These heuristics distinguish between two types of
browsers which are supported:

- **Evergreen browsers**, those which have adopted a continuous update policy
  and generally keep their users up-to-date automatically.
- **Non-evergreen browsers**, which have more complicated support patterns that
  may prevent users from updating regularly.

Depending on which category a browser falls into, different rules will be
applied. This will allow us to effectively support browsers which release
frequently, such as Chrome, and browsers which do not, such as Safari.

## Existing Policy

The existing policy is listed here for documentation purposes.

- Desktop

    1. Google Chrome, Mozilla Firefox, MS Edge, Mac Safari

        - The current and previous stable releases are supported.

    2. Internet Explorer

        - Only Internet Explorer 11 is supported.

- Mobile

    1. All

        - All mobile browsers may work but are not explicitly supported.

Any browsers not listed here may work but are not explicitly supported.

## Proposed policy

In the new support policy, we define two categories of browsers:

1. **Evergreen browsers**, which are browsers that:

   * Support using the latest version on all supported platforms (e.g. you can
     use the latest Chrome an any supported version of Windows, macOS, Android,
     etc. The browser is versioned independently from the operating system).
   * Automatically update whenever a new version is available.

2. **Non-evergreen browsers**, which are any browsers which do not meet the
   criteria of evergreen browsers. For instance, Safari is a supported browser
   whose version is tied to the version of macOS and iOS that users use. As
   users will often wait to update their operating system, many users lag behind
   on the version of Safari they are using, so it cannot be considered
   evergreen.

Using these categories, Ember will support the following browsers:

- Evergreen
  - Desktop
      1. Google Chrome/Chromium
      2. Mozilla Firefox
      3. Microsoft Edge
  - Mobile
      1. Google Chrome
      2. Mozilla Firefox

- Non-evergreen
  - Desktop
    1. Safari
  - Mobile
    1. Safari

Any browser which is not listed here may work, but is not explicitly supported.
In addition, in order to recategorize browsers as evergreen or non-evergreen, a
new RFC must be submitted to update the support policy.

So for instance, if Safari were to begin automatically updating its version
independently of macOS/iOS versions, and thus fulfill the criteria for evergreen
browsers, it would still be treated as a non-evergreen browser until this policy
is updated. This will prevent sudden and unexpected changes in the support
matrix for Ember users.

### Evergreen browsers

Ember and Ember Data will support **all** versions of evergreen browsers which
meet either of the following criteria:

1. It is the latest version of the browser.
2. It has at least **0.25%** marketshare usage, based on
   [statcounter](https://gs.statcounter.com/).

This policy will allow Ember to generally support the most recent 2-3 versions
of the browser, and account for versions which may become temporary transition
points. For example, when MS Edge made the switch to the new Chromium-backed
version, the last non-Chromium version saw higher usage for a slightly longer
period of time as the update process was not as standard, and required an
explicit opt-in. These transitionary periods are important, but generally
speaking temporary, and once the usage drops below the threshold Ember can drop
support at that point.

The supported versions of browsers are determined at the time of each _minor_
release of Ember and Ember Data. In the release blog post, we will specify
which versions of browsers are supported for this minor version, to keep users
informed as usage changes.

_Within_ a minor version, support works as follows:

1. Support for a given browser version is _never_ removed.
2. Support is _added_ for any new versions of a browser that have been released
   while the minor is still supported.

So for instance, if we were to release Ember 4.4 with support for Chrome 92, 93,
and 94, then bugfixes and security fixes for those versions of Chrome will be
supported until 4.4 is no longer an LTS, following the LTS support matrix. In
addition, if Chrome were to release version 95 while 4.4 is still actively
supported, then 4.4 will also support Chrome 95.

### Non-evergreen browsers

Ember and Ember Data will support the following versions of non-evergreen
browsers:

- Desktop
  - Safari: 14
- Mobile
  - Safari: 14

These versions will continue to be supported until support is explicitly dropped
via a new RFC, and dropping support for any version of these browsers will
require a new **major** release.

In addition, all new major releases of these browsers will be supported as they
are released. Like with evergreen browsers, support is retroactively added to
any minor version of Ember which is still supported.

#### Note on the versions chosen

The versions chosen here are the latest versions of Safari, which having just
been released this year, do not have 100% adoption. As such, there are still a
significant number of users who are using previous versions of Safari. However,
by the time this support policy is enacted and a new major version of Ember is
released, this number will likely have shifted significantly.

In addition, since Ember will not drop support for any version of Safari until
the next major release after this policy is enacted (v5), this decision strikes
a balance between enabling usage of new APIs and patterns, while still providing
support for users who are stuck on older versions of Safari. While we are
leaping ahead now, in the coming years the majority of users will upgrade beyond
this version, and eventually we will be supporting a much older version of
Safari than is actually used in practice.

Supporting Safari 14 and above means that Ember can safely use the following JS
features:

- The BigInt data type.
- Creating custom instances of EventTarget.
- Logical assignment operators.
- Public class fields.

In its own internal code, without needing to worry about polyfills. Of course,
polyfills can still be provided by Ember applications, and may continue to work -
they are just not explicitly supported.

## What support means

This policy governs two major aspects of Ember and Ember Data:

1. When the framework adopts new browser features
2. How the framework responds to bug reports

Ember and Ember Data _may_ introduce new browser feature usage in any _minor_
version release of Ember, provided the feature is supported in all browsers that
are supported in this policy at the time the the version in released.

Ember and Ember Data _may_ introduce new browser feature usage in any _patch_
version release of Ember, provided the feature is supported in all browsers that
were supported in this policy at the time the _minor_ that the patch is applied
to was released. In other words, no breakage can or should occur due to new
browser feature usage in _any_ patch release, and if it does, it is a bug.

For bug reports, support is determined by combining our existing Ember version
support policy with the browser versions that were supported at the time an
Ember version was released. This means Ember will work as time and resourcing is
available to fix any browser specific issues that occur in:

1. The current stable and LTS releases of Ember and Ember Data
2. For browsers that were supported by those versions when they were relased

For the previous LTS release, only security bugfixes will be supported,
following the existing LTS policy.

## Future changes to the policy

This policy is not meant to be immutable. Over time, new browsers could be added
to the support matrix, and explicit exceptions could be added for "critical
internet infrastructure".

For example, it was well known that Google's search crawler used a very old
version of Google Chrome for a very long time. It no longer does and is
regularly updated, but during that time frame it was important to support that
version of Chrome. Likewise, it was very important to support IE11 for a very
long time since it had a large usage percentage, especially in corporate
environments. While none of these cases are known to exist at the time of this
writing, if one should arise it may be added explicitly via RFC.

Future RFCs may amend this policy in a strictly additive way without requiring
a major version bump in Ember. Newly supported browsers will begin being
supported in the next minor version of Ember after the updated support policy is
implemented.

Future RFCs that amend this policy to _remove_ support for a browser will
_require_ a major version of Ember to implement.

## Implementation

This policy drops support for a major browser, and therefore can only be
implemented with a major version bump.

## How we teach this

- A blog post explaining the policy with a link to this RFC will be posted.
- Browser support policy will be linked in repo
- Browser support policy will be added to the website
- Browser support policy will have an announcement in the Ember Times

## Drawbacks

- The proposed policy makes Safari the new "lowest common denominator" browser,
  replacing IE11. Since the supported versions of Safari will continue to be
  supported indefinitely, we will not be able to use any new browser features
  added afterwards without another major version. In particular, the spec for
  `WeakRef` has finally been stabilized, and this is something we'll likely want
  to make use of in the not-to-distant future.

  While this is true, it does not impact the immediate roadmap for Ember in the
  next few years. When the time comes, we can update our browser support policy
  again, and release a new major version.

## Alternatives

- Keep things as they are.
- Document the existing policy but do not change it.
- Support fewer browsers, for a shorter amount of time.
- Support more browsers, for a longer amount of time.
