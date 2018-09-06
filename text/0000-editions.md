- Start Date: 2018-09-04
- RFC PR:
- Ember Issue:

# Editions

## Summary

Introduce a new concept: editions. Every few years, we will declare a new edition of Ember that bundles up accumulated incremental improvements into a cohesive package.

Editions give us an opportunity to bring our documentation and marketing up-to-date to reflect the improvements we’ve made since the previous edition. They are also the time to synchronize Ember and the wider ecosystem, including updating default settings, blueprints, addons, and more.

Editions also help ensure that Ember users are getting a cohesive, polished experience. The right time to declare a new edition is when:


- A significant, coherent set of new features and APIs have all landed in the stable channel.
- Error messages and the developer ergonomics of those new features have been fully polished.
- Tooling (the Ember Inspector, blueprints, codemods, etc.) has been updated to work with these new features.
- API documentation, guides, tutorials, and example code has been updated to incorporate the new features.

## Motivation

Ember releases a new, stable version every six weeks. For existing users, this drumbeat of incremental improvement is easier to keep up with than splashy, big-bang releases.

However, this steady stream of small improvements comes with some downsides.

- For Ember users, it’s hard to keep track of evolving best practices, and understand how new features are supposed to work together.
- For non-Ember users, it’s not obvious what has changed and whether Ember is worth another look.
- For Ember contributors, it’s not clear when to update documentation and marketing, because there’s always more features in the pipeline.

This RFC proposes a new release concept called “editions”, which provides a lens through which to view these accumulated changes. It also marks an opportunity to synchronize the wider ecosystem with the latest changes to Ember, including updates to defaults, documentation, addons, and more.

## Detailed design

Editions touch several parts of the Ember ecosystem, and generally tie together a set of incremental changes with a unifying theme. Most of these changes should come from existing community processes (i.e. RFCs), with editions serving as a higher level conceptual grouping.

### Editions vs. Releases
Editions are not separate releases, but rather a specific range of releases belong to an edition. The current edition will be open-ended - it’s terminating release is determined by whenever the next edition is declared.

### Proposing Editions
Each edition is proposed via a RFC, which should describe in detail the change in programming model and how to communicate that with users.

### Features
Editions mark an opportunity to focus on shipping a set of thematically related feature work. For example, the first edition, Ember Octane, focuses on “performance and productivity”. It includes a few larger features as well as several smaller ones, all focused on delivering that theme. Note that editions are backwards compatible as well - they are decoupled from major semver releases.

### Documentation
Documentation is often a developer’s first encounter with a framework, and editions offer a chance to revisit that experience with fresh eyes in light of that edition’s theme and programming model. The guides, tutorials, and related documentation should be updated where needed to address the themes of the edition, and to ensure that all the feature work is fully documented.

### Defaults
Editions offer an opportunity to reassess the defaults of the Ember ecosystem. For example, it’s a good time reassess the set of addons installed by default in the CLI new app blueprint. Or to switch opt-in behaviors to opt-out behaviors (keeping semver compliance in mind). But it’s not limited to the CLI and Ember.js only - addons are encouraged to shift their default configuration and functionality to align with the edition’s goals at the same time (where appropriate).

### Addons
A core concept of editions is that they offer a chance to refocus the community as a whole around a set of shared goals. That includes the rich addon ecosystem that Ember offers. Addon authors are encouraged to consider the goals of upcoming editions, and update their addons - where appropriate - to align with those goals.

### Marketing
Because of Ember’s strict semver compliance, Ember’s major versions can often seem like lackluster affairs: simply removing deprecated code. Editions provides an opportunity to accumulate the hundreds of tiny improvements into a single large marketing and branding opportunity that can build interest and hype around the framework.

## What Editions are Not

It’s important to note that editions are not:

- Semver major releases. Ember uses semver for it’s versioning strategy, and major releases are usually just removing features deprecated in previous minor versions. Editions allow us to decouple the technical details of semver compliance from the overall marketing and developer experience efforts


## How we teach this

Editions are, in large part, a teaching tool. They serve as a rallying point for updates to the documentation and overall developer experience.

To new Ember users, they may not see many references to an edition by name during the learning process, but the documentation they consume, the features they use, the marketing they see, etc., will all be focused through the lens of the current Ember edition.

For existing users, editions become a way to mark the larger progress of Ember, especially for those users that don’t follow the minor releases closely.

For contributors, addon authors, and core team / subteam members, editions are a rallying point to focus a consolidate group effort around, and a tool to help ensure a cohesive developer experience.

## Drawbacks

Editions introduces more conceptual overhead to understanding Ember - now, in addition to individual releases & release channels, there’s a third vehicle for understanding Ember milestones, which does not necessarily correlate to major semver versions as some developers might expect.

## Alternatives

- Don’t do editions at all
- Editions are purely marketing focused, i.e. the now withdrawn Marketing Releases RFC

## Unresolved questions

- What to call these? “Editions” is a good candidate, but might introduce confusion vs. releases (as evidenced by the comment thread on the 2018 Roadmap RFC). “Epochs” was also proposed as it better captures the duration aspect (“edition” sounds like a moment in time, whereas this concept spans multiple releases), but is confusing for most people to pronounce and might present difficulties for non-native English speakers. Perhaps there’s a better word to adopt for this concept?

## Prior Art

Much of the edition concept outlined in this RFC borrows heavily from the Rust language community's concept of editions (previously called epochs). For more background, check out the [RFC itself](https://github.com/rust-lang/rfcs/blob/master/text/2052-epochs.md) as well as [the discussion thread](https://internals.rust-lang.org/t/evolving-rust-through-epochs/5475)
