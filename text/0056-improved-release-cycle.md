---
Start Date: 2015-10-02
RFC PR: https://github.com/emberjs/rfcs/pull/56
Ember Issue: (leave this empty)

---

# Refining the Release Process

Ember balances a desire for overall stability with a desire for continued improvements using a two-pronged approach:

* General adherence to Semantic Versioning, which means that we don't 
  make breaking changes to public, documented APIs except when the 
  major version changes.
* A rapid release cycle that allows us to ship additive changes to the
  framework on a regular, digestible basis.

Since Ember 1.0, we have refined this approach:

* All new public APIs are added using feature flags onto the master
  branch. Feature flagged features are not included in beta or release
  builds until they are "Go"ed by the core team.
* We avoid breaking heavily used but private APIs in minor versions.
* When we feel we must break a private API that is heavily used, we
  use a two-step deprecation approach: deprecate the private API in
  one release and remove it in a subsequent release, once apps and
  add-ons have had an opportunity to upgrade.
* When we plan to make breaking changes in a future major release,
  we first deprecate the changes in a previous minor release.
* We never deprecate features until there is an already-landed
  transition path to a new approach whose feature flag has already
  been "Go"ed.

And finally:

* A major release does not introduce any new breaking changes that
  were not previously deprecated. Major versions simply remove
  deprecated features that already landed.

Ember 2.0 is the first major release cycle where we have followed these refinements; this document is an attempt to outline some additional refinements that we might adopt going forward.

## Benefits of the 1.x Model

* New features are added predictably, and it's relatively easy to
  follow the list of new APIs that are under development, and where
  they are in the process.
* There is little pressure for contributors to land a feature 
  prematurely, because missing a release deadline isn't
  catastrophic–there will be another train six weeks hence.
* We have a lot of very good automation tools that keep the trains
  running–commits can be (mostly) automatically backported to the 
  current beta or release version.
* Upgrading Ember itself from version to version is typically a quick 
  process, except when private APIs are in use. We aim for upgrades to
  be possible to slot into existing product sprints, and the nature of
  the process means that we tend to hit this goal for most users.
* Upgrading Ember across a number of versions is typically pretty
  straightforward, at least in theory.

In total, this process provides a way for us to clearly message medium-term changes in a way that helps you make the changes predictably and as mechanically as possible.

The process of getting from *here* to *there* is a series of incremental releases with deprecations, which gives you a trail of breadcrumbs to follow as things change.

## Problems with the 1.x Model

While the approach we're using has provided a lot of benefits, there are a number of areas that could still use improvement:

* While it is in theory possible to upgrade only once every few 
  releases, there is no guidance about exactly how to do that, and
  little clarity about how many releases we plan to support with
  security fixes. (Because of the two-step deprecations of heavily 
  used  private APIs, it is in practice important to go through each
  intermediate release to clear deprecation warnings before
  proceeding.)
* While SemVer guarantees apply to public APIs, many addons are forced
  to use private APIs as part of experiments. These experiments are a
  crucial part of the evolution of the Ember ecosystem, and the Ember
  1.x series has had a fair bit of churn in these APIs.
* While the SemVer guarantees apply to Ember proper, they do not apply
  to parts of the blessed experience that have not yet reached 1.0.
* While the SemVer guarantees promise that your code will continue
  working, they do not address changes to idiomatic Ember usage, which
  can change over time. In practice, this means that there can be
  churn in the experience of using Ember without actual breakages.
* While deprecations technically don't force you to change anything,
  in practice clearing deprecations is a part of the upgrade process.
  A constant stream of deprecations, like in the lead-up to Ember 2.0, 
  can feel almost as bad as breaking changes.
* In the lead-up to Ember 2.0, a desire to remove as much cruft as
  quickly as possible led to a need to land new features with much
  more urgency than usual.

In total, these problems introduce churn in the experience of using Ember. In practice, things like moving to ES6 modules, moving to Ember CLI, and the changes in Ember Data have made the experience of "keeping up" more frenetic than we would have liked.

Because Ember releases a new version every six-weeks, it's easy to associate the overall churn with the rapid pace of releases.

## Non-Goals of the Improvements

The release process does not attempt to change the overall pace of change, but rather to make changes more predictable, easy to track, and easy to upgrade to.

The six-week cycle can incidentally affect the pace of change, because it means that large changes usually need to be broken up into pieces that can land a bit at a time. However, in practice this speeds up ecosystem-wide adoption of the entire feature, because people do not find themselves stuck behind a big-bang change that they can't schedule the time to upgrade to.

A recent survey of the Ember ecosystem, which had close to 1,000 respondents, showed that the vast majority of Ember users are using one of the past three versions of Ember.

## Proposal: LTS Releases

In theory, it's possible to upgrade every few releases, instead of every release. This has a few drawbacks:

* Because of the two-step deprecation process for heavily-used
  private APIs that we want to remove, it is in practice necessary
  to go through all intermediate releases in order to catch possible
  deprecations.
* We currently don't have any official policy about which exact
  releases we backport security patches to, other than a promise
  that we will always backport to the previous released version.
* Since different people upgrade at different rates, it's hard for
  add-ons and other parts of the Ember ecosystem that are not
  bound by the same SemVer guarantees to know which versions to
  continue to support.

**I propose that every 4 releases is considered a "Long-Term-Support (LTS) release" . With the six-week cycle, that means every 24 weeks, or roughly twice per year.**

This means:

* We will only remove heavily used private APIs if they were 
  deprecated in a previous LTS release. This means that
  if a feature is deprecated in 2.3, the first LTS release that 
  the deprecation will appear in is 2.4, and it can therefore be 
  removed in 2.5.
* We will provide release notes for each LTS release that
  roll up the changes for the releases it includes, including new
  deprecations and new features.
* We will use the LTS releases to provide better big-picture 
  messaging on the goals of any deprecations and changes to
  idiomatic Ember.
* Security fixes will always be backported to the most recent
  LTS release.
* We will encourage the Ember ecosystem to maintain support for
  the LTS releases, and lead by example with our own
  projects that have not yet reached SemVer stability. Ideally, this
  will give more of a voice to people who are upgrading less 
  frequently.

This means that people who want to stay on the latest and greatest can continue to upgrade every six weeks (with the same SemVer guarantees we've come to expect), and people who want to upgrade less frequently can do so.

In practice, since these releases still abide by SemVer, upgrading from LTS release to LTS release should not be significantly more work than upgrading along the six-week release cycle.

Upgrading less frequently will mean, of course, that you would need to wait to take advantage of new features, and experience less gradual changes to idioms. It will also mean that every upgrade will come with a bigger bundle of deprecations to clear.

> It is important for us to keep an eye on the situation to see whether less frequent updates result in people getting left behind.

## Proposal: Svelte Releases and Major Releases

Another problem worth addressing is that, as Ember gradually deprecates old idioms to make way for new ones, SemVer guarantees require that we continue shipping deprecated features until the next major release.

This has two related problems:

1. Ember users who are not using deprecated features need to continue
   shipping deprecated code, which increases both code bloat and
   an opportunity to accidentally slip back into older idioms.
2. Ember itself needs to continue maintaining support for
   deprecated features in its internals, which, over time, results
   in cruft that impacts our ability to improve Ember.

However, we also need to be cognizant of the fact that changes to Ember idioms take time to be reflected in online materials, so it's important for snippets copy-and-pasted from tutorials to continue to produce deprecation notices for some time.

In general, this is the question of how to "garbage collect" cruft in the framework gradually and with minimal impact.

Leading up to the 2.0 release, we thought we would address this issue with periodic "cruft removal" major releases. Every so often, we would issue a major release with the primary purpose of clearing out accumulated cruft. Minor releases could create deprecations, but not purge their associated code.

Unfortunately, because of the fact that **Ember does not generally deprecate features without a clear transition to something else**, this meant that the 2.0 release became a critical release for adding new features as well. In the run-up to 2.0, we felt a higher degree of urgency to add new features in the programming model to replace ones we expected to want to remove early in the 2.x series.

The goal of the train release model is to eliminate big-bang releases and the attendant stress on releasing particular features by a given date, and the 2.0 release has been far too disruptive to that goal.

In the 2.x cycle, I propose a few enhancements:

1. Ember itself will more clearly mark deprecated features in a
   similar way that it marks new features, including with the
   release it was deprecated in.
2. Ember CLI will support "svelte builds", which strip out
   deprecated features.
3. In development mode, Ember CLI will convert deprecated features
   into errors, to ensure that people running svelte builds can still
   get clear messages when using code that was designed for earlier
   builds, including addons.
4. We will still use major releases to remove built up cruft,
   especially deeply intertwined cruft, but the svelte releases
   should take the pressure off of the major release timeline.

**The 1.x release cycle helped us establish an orderly process for adding features; this proposal establishes a more orderly process for removing them.**

## Proposal: Plugin APIs

Since the release of Ember 1.0, we have worked on refining the public APIs while maintaining stability. However, those public APIs do not cover all possible use-cases, and add-ons have cropped up to fill the gaps.

Unfortunately, this has placed a heavy compatibility burden on add-on authors who want to maintain stability in their public APIs even as versions of Ember have changed the private APIs they rely on.

In practice, the costs of the six-week release cycle weigh most heavily on add-on authors, who are often forced into using private APIs, but still want to keep their add-ons working with every release.

The canary and beta cycles help to ensure that popular add-ons work by the time the release version comes out, but only because add-on authors keep a close eye on the beta releases and absorb the churn on behalf of their users.

**I propose that as of Ember 2.0, any use of a private API in a plugin is considered a bug in Ember to be fixed.**

That doesn't mean that add-on authors should never use private APIs: to the contrary, use of private APIs when no other choice is available helps us discover what APIs are missing.

But a major goal of the 2.x series of Ember should be to identify ways to extend the stability promises that Ember offers to application authors to add-on authors.
