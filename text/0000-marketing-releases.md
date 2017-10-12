- Start Date: 2017-10-09
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Add "named releases" to allow for easier marketing efforts while simultaneously maintaining Ember's strong semver guarantees.

# Motivation

Ember's commitment to semver is valuable. The practice of introducing new features in minor versions and major versions only removing deprecated functionality makes for a straightforward and predictable upgrade path.

Unfortunately, this is not the norm throughout the JavaScript community, despite the wide usage of semver version numbering via tooling like npm. This leads to a marketing problem, as evidenced by some of the reaction to the Ember 3.0 announcement.

For most other projects, a semver major bump is a big marketing event. Culturally, the expectations are to see great new features. For Ember, semver major bumps are comparatively lackluster - simply removing functionality that you probably have stopped using anyway.

This may lead developers who may not fully understand our semver practices to assume that the apparently lackluster 3.0 announcement is indicative of a lackluster project (when in reality, it's the opposite).

This RFC proposes that we attempt to decouple semver version numbers from our marketing efforts, and adopt a different set of "marketing releases" which can be used to generate interest around new features. The goal is to find a solution that allows us to keep our strong semver guarantees while also allowing for effective marketing of Ember's new features and advancements.

# Detailed design

> Note: this is one possible approach to marketing releases - others are discussed in the Alternatives section below

Periodic minor releases are identified as containing noteworthy or marketable new features, and flagged as a "named release". Names would be chosen in alphabetical order to help communicate order, and should be simple and memorable.

The core team must reach a consensus on whether or not to "name" a minor release, but anyone in the community may suggest it if that release is not already under consideration.

> Note: this section was left intentionally light on process. One goal here is to avoid burdening any contributors with significant additional process. If we move forward with names releases and discover this process to be too loose, we can propose additional refinements. Premature optimization and all that ...

# How We Teach This

This is actually primarily a teaching & marketing tool. This would help users discover new features and advancements in Ember. Blogs and newsletters would transition to referencing release names when discussing new and upcoming features.

# Drawbacks

I've split this section into two parts: drawbacks to the idea of marketing releases _in any form_, and drawbacks to the specific proposal from above to attach a name to certain minor releases deemed "noteworthy enough".

## Drawbacks to marketing releases in general

* Identifying releases to "market" might be tricky or contentious, i.e. "3.2 has features X & Y - is that enough to justify a 'marketing' release?"
* Might not be helpful enough. With the size and history of the Ember community, the developer culture may be entrenched deeply enough in semver release numbers to change course to thinking of marketing releases.

## Drawbacks to "named releases" specifically

* Alphabetical named releases help convey ordinality, but not as readily as numbers might
* The alphabetical approach might be difficult with certain letters (i.e. a named release for "X")
* Named releases are only part of the solution - the rest is the actual marketing of the releases. Without effective marketing, named releases are deadweight process

# Alternatives

* We could not do marketing releases at all
* One minor variant of the above proposal, suggested by @acorncom: named releases correspond 1:1 with LTS releases.
* Rather than "naming" the marketing releases, suggested by @runspired: add another digit to the version number that is solely for marketing: <Marketing>.<Major>.<Minor>.<Patch>

# Unresolved questions

N/A