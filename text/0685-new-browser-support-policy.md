---
Stage: Accepted
Start Date: 2020-11-28
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): All
RFC PR: https://github.com/emberjs/rfcs/pull/685
---

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

## Proposed policy

In the new support policy, Ember will support the following browsers:

- Desktop
  1. Google Chrome
  2. Mozilla Firefox
  3. Microsoft Edge
  4. Safari
- Mobile
  1. Google Chrome
  2. Mozilla Firefox
  3. Safari
- Testing
  1. Headless Chrome
  2. Headless Safari

Any browser which is not listed here may work, but is not explicitly supported.

In addition, these browsers have been categorized into _evergreen_ and
_non-evergreen_ browsers, which have different support policies. This categories
have been chose arbitrarily, based on the current state of browsers and their
support policies.

- Evergreen
  - Desktop
    1. Google Chrome
    2. Chromium
    3. Mozilla Firefox
    4. Microsoft Edge
  - Mobile
    1. Google Chrome
    2. Mozilla Firefox
  - Testing
    1. Headless Chrome
    2. Headless Safari

- Non-evergreen
  - Desktop
    1. Safari
  - Mobile
    1. Safari

This categorization will _not_ change without an additional RFC, even if the
browsers themselves make significant changes to their own release process or
support system.

#### Heuristics for browser categorization

As mentioned above, the evergreen and non-evergreen categorization is ultimately
arbitrary. However, there were some heuristics which were used to categorize the
supported browsers at the time this RFC was made:

1. **Evergreen browsers** are browsers that:

   - Support using the latest version on all supported platforms (e.g. you can
     use the latest Chrome an any supported version of Windows, macOS, Android,
     etc. The browser is versioned independently from the operating system).
   - Automatically update whenever a new version is available.

2. **Non-evergreen browsers** are any browsers which do not meet the
   criteria of evergreen browsers. For instance, Safari is a supported browser
   whose version is tied to the version of macOS and iOS that users use. As
   users will often wait to update their operating system, many users lag behind
   on the version of Safari they are using, so it cannot be considered
   evergreen.

### Evergreen browsers

For a given Ember and Ember Data minor release, the minimum major version
supported for a given browser is determined using the following formula.

- Whichever browser version is greater/more recent out of:
  1. The lowest/least recent version that fulfills any one of these properties
    - It is the latest version of the browser.
    - It is the latest LTS/extended support version of the browser (such as Firefox ESR).
    - It has at least **0.25%** of global marketshare usage across mobile and
      desktop, based on [statcounter](https://gs.statcounter.com/).
  2. The minimum version supported in the previous release

Within a major version of a browser, the latest patch release is the only
release that is supported.

This policy has the following attributes:

- It allows us to generally support the most recent browser versions, which
  are typically used for some time before everyone upgrades
- It allows us to support browser versions that become temporary "speedbumps"
  that take longer for users to update, such as the last non-Chromium version of
  MS Edge.
- It prevents backsliding, once a version is no longer supported, it never
  becomes supported again.

Above all else, this policy is easy to communicate - for every minor release, we
calculate the minimum supported version, and then communicate that we support
all versions >= that version.

It is important to note that this policy means that each minor release _may_
drop support for some major versions of evergreen browsers.

### Non-evergreen browsers

Ember and Ember Data will support all major versions greater than or equal to
the following version for non-evergreen browsers:

- Desktop
  - Safari: 14
- Mobile
  - Safari: 14

These versions will continue to be supported until support is explicitly dropped
via a new RFC, and dropping support for any version of these browsers will
require a new **major** release.

Within a major version of a browser, the latest patch release is the only
release that is supported.

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

In order to support this policy while keeping all of our tooling on the same
page, Ember.js itself will add automation to calculate the current minimum
supported version of browsers on its main branch, and publish them in an
accessible format. Each time we branch a release, these versions will stop
updating, so they will effectively be locked in.

These versions will be used for a variety of use cases, such as generating
documentation and release blog posts (see below), and generating the default
`config/targets.js` for new Ember apps and addons.

### Implementation timeline

This policy drops support for a major browser, and therefore can only be
implemented with a major version bump.

## How we teach this

This will be an ongoing process, since the minimum browser versions supported
change over time. The most important thing here is that users can easily
determine whether or not a given version of a browser is supported for a given
Ember release.

The following are the ways we will communicate this:

- For the release blog post for a minor version, we'll include a table which has
  the list of every supported browser, along with the minimum supported major
  version of that browser for the release
- On the [releases page](https://emberjs.com/releases) of the Ember.js website,
  for each of the listed releases, we will include a table of the supported
  versions major browsers for that release.
- The browser support on the releases page for the Stable and LTS branches will
  be linked in the Ember.js README.

This documentation will be supported by the minimum supported versions that are
published in the Ember.js package (see above), which will allow them to be
mostly automated. The supported browser table could look like the following
(generated using today's usage stats):

### Supported Browsers

#### Desktop

| Chrome | Edge | Firefox  | Safari |
| ------ | ---- | -------- | ------ |
| 83     | 18   | 78 (ESR) | 14     |

#### Mobile

| Chrome | Firefox | Safari |
| ------ | ------- | ------ |
| 87     | 83      | 14     |

#### Headless

| Chrome | Firefox  |
| ------ | -------- |
| 87     | 78 (ESR) |

### "How we determine support" page

In addition, the technical details of the support policy should be documented on
the Ember website, in a less prominent position (e.g. a link from the supported
versions table titled "how do we determine browser support?"). This could use
the following text:

Ember supports the following major browsers:

- Desktop
  1. Google Chrome
  2. Mozilla Firefox
  3. Microsoft Edge
  4. Safari
- Mobile
  1. Google Chrome
  2. Mozilla Firefox
  3. Safari
- Testing
  1. Headless Chrome
  2. Headless Safari

Other browsers may work with Ember.js, but are not explicitly supported. If you
would like to add support for a new browser, please [submit an RFC or RFC issue for discussion](https://github.com/emberjs/rfcs)!

We determine support on a browser-by-browser basis. Browsers are categorized as
either **evergreen** or **non-evergreen**. The categorization is as follows:

- Evergreen
  - Desktop
    1. Google Chrome
    2. Chromium
    3. Mozilla Firefox
    4. Microsoft Edge
  - Mobile
    1. Google Chrome
    2. Mozilla Firefox
  - Testing
    1. Headless Chrome
    2. Headless Safari

- Non-evergreen
  - Desktop
    1. Safari
  - Mobile
    1. Safari

For evergreen browsers, the minimum version of the browser that we support is
determined at the time of every minor release, following this formula:

- Whichever browser version is greater/more recent out of:
  1. The lowest/least recent version that fulfills any one of these properties
    - It is the latest version of the browser.
    - It is the latest LTS/extended support version of the browser (such as Firefox ESR).
    - It has at least **0.25%** of global marketshare usage across mobile and
      desktop, based on [statcounter](https://gs.statcounter.com/).
  2. The minimum version supported in the previous release

To simplify, the supported version either moves forward or stays the same for
each release based on overall usage and LTS/current release versions.

For non-evergreen browsers, support is locked at a specific major version, and
we support all major versions above that version:

- Desktop
  - Safari: 14
- Mobile
  - Safari: 14

Within a version of a browser, we only support the most recent patch release.

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
- Drop IE11 but not add more definition to support (keeping it as a case by case
  determination by the core teams)
