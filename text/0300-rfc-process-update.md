---
tags: ember, ember-cli
---

- Start Date: 2018-02-04
- RFC PR: https://github.com/emberjs/rfcs/pull/300
- Ember Issue: (leave this empty)

# RFC Process Update

## Summary

Introduce the idea of Core Champion to the Ember.js RFC process, merge Ember.js and Ember CLI RFCs repositories,
and add metadata to make cross-project RFCs easier.

## Motivation

The Ember community has been using the Request For Comments (RFC) process to great effect over the last few years.
The RFC process has proved invaluable.
Proposals by both Core and community members are discussed and refined,
with the result coming out much stronger.
Other projects like React also adopted an RFC process, lending external validation to this idea.

Following its initial implementation, shortcomings of the RFC process as well as new requirements were identified,
which we will try to address in this proposal.

### Confusion about emberjs/rfcs and ember-cli/rfcs

The Ember project currently has two separate RFC processes for Ember.js and Ember CLI.

This leads to confusion because community members must keep track of two different repositories in order to monitor progress.
The submitter must consider these differences during the lifecycle of the RFC.

There are also some cases where it is not clear exactly where the RFC should be submitted.
This can also lead to multiple RFCs being written, one for each repository.
While multiple RFCs might be warranted in some cases,
it should be a choice instead of a limitation of the current processes.

### Process does not cover sufficient areas of development

RFCs to emberjs/rfcs and ember-cli/rfcs concern themselves mostly with features or deprecations to ember.js and ember-cli respectfully, with some ember-data proposals in emberjs/rfcs.

Once again taking inspiration from Rust, they have experimented expanding the types of RFCs proposed
[crates.io default ranking](https://github.com/rust-lang/rfcs/blob/master/text/1824-crates.io-default-ranking.md)
[default URLs for documentation](https://github.com/rust-lang/rfcs/blob/master/text/1826-change-doc-default-urls.md).
The latter one, for example, would have been a useful RFC when designing the new [API docs](https://github.com/ember-learn/ember-api-docs).

### Lingering RFCs

The emberjs/rfcs repository currently has [54 issues](https://github.com/emberjs/rfcs/issues) and [62 PRs](https://github.com/emberjs/rfcs/pulls) open, and ember-cli/rfcs has [183 issues](https://github.com/ember-cli/ember-cli/issues) and [49 PRs](https://github.com/ember-cli/ember-cli/pulls).
A significant percentage of these have not been active in the recent past.

We have kept PRs and issues open so people could more easily find the discussions,
but this has instead given a negative feeling of staleness, as RFCs linger open without new feedback.

## Detailed design

To address the issues raised in the “Motivation” section, I will propose changes to the current process.

### Implement Core Champion

To make sure that RFCs receive adequate support from the team, Ember CLI has implemented the idea of a champion associated with each RFC.
The goal is that in seeking a champion from the team,
the RFC author starts a dialogue with the team and gets some early feedback.
The champion is then responsible for representing the RFC in team meetings, and for shepherding its progress.

This will be imported into emberjs/rfcs.

### Improve the lifecycle of RFCs

To address the triaging and inactivity problem of RFCs, I introduce the concept of FCP to close.

Closing an RFC should be viewed as another triaging tool, not as a rejection of the RFC.
Sometimes a rewrite of an RFC would be so fundamental that it would benefit of a fresh discussion in a new thread.
Sometimes the original author is no longer active (Champions should help here as well),
and someone else might want to take over the work in a new RFC.
Sometimes the timing might not be right, or the feature might have been addressed some other way, and yes,
sometimes it might be something that is not aligned with the team's values for the project.

A good example of this is the [Named Blocks RFC](https://github.com/emberjs/rfcs/blob/master/text/0226-named-blocks.md#motivation),
which lists in the motivation section previous attempts at similar ideas.

Like the FCP to merge process, once an RFC is marked as FCP to close there will be a period of one week where people can raise new concerns.
After that period of one week, the respective team will review and close the RFC or extend the period for another week.

### Merge ember-cli/rfcs into emberjs/rfcs

Part of removing confusion about the process is having a central repository for all Ember RFCs.

To achieve merging ember-cli/rfcs into emberjs/rfcs the following will be done:

- Remove active/completed concepts from ember-cli/rfcs
  - Add front matter
  - Copy both active and completed RFC files into `text` of emberjs/rfcs
- Transfer active PRs and Issues to emberjs/rfcs
- Archive ember-cli/rfcs

There are some concerns about links breaking when we move the files to emberjs/rfcs,
but given the fact that ember-cli/rfcs had the concept of active/completed by moving the files into different folders,
links were already being broken.

The ember-cli/rfcs do not need name or numbering changes, as there is currently no duplicated name.
Going forward, the numbering should be unified by virtue of having a single repository.

### Open process to more types of RFCs

To build on the previous point, the front matter that is to be added to RFCs will make it easier to have RFCs for other areas of the project,
like the design of the web resources, Guides content, marketing content, or features for core addons.

Given that Ember is [organized into teams](https://emberjs.com/team/), with each team being responsible for certain projects,
the RFC front matter should be oriented towards those teams.

A list of the teams and respective projects should be added to the instructions,
possibly with the addition of per-team instructions on specifics of the project.
Additional templates might be created as well, such a design work template.

### Provide more guidance for post-RFC

At the moment it is not clear for some contributors what happens once an RFC is merged.
How and when should work to implement an RFC be done?

## How We Teach This

To ensure that contributors are updated on the RFC process and the process is clear,
the documentation should be improved in a couple of ways.

The README should be updated to include information about Champions,
how they are assigned, what they do.
The README should include a description of the lifecycle of RFCs with the new "FCP to close" state,
and instructions on how to proceed with the implementation of the RFC once it is accepted.

## Drawbacks

### Adjustment period

There are active RFCs in ember-cli/rfcs. Moving these discussions would be onerous, so they should be kept there until completion, and no new RFCs accepted.

### Permalinks to ember-cli/rfcs proposals

Moving the RFC files from ember-cli/rfcs (active or completed) to emberjs/rfcs can be seen as a breaking change, and could lead to someone linking to ember-cli/rfcs and then the RFC being updated in emberjs/rfcs. However, ember-cli/rfcs already suffers from a linking problem due to the active/completed folders, as RFCs need to be moved from one to the other even after being accepted.
This could be mitigated by introducing a warning in the RFC text directing people to the new source.

## Alternatives

None at the moment.

## Unresolved questions

### How do Champions get assigned to an RFC?

More enterprising authors will likely reach out to team members on their own,
but the added burden of having to seek a Champion might discourage people from authoring RFCs.

The various teams could agree to review new RFCs during the weekly calls and assign Champions?

---

## Glossary

- **RFC**: Request For Comments. The process by which a proposal is discussed by the community and then approved by an Ember team.
- **FCP**: Final Comment Period. Period of one week at the end of which an RFC is to be accepted or rejected by an Ember team. Extended in periods of one week if new concerns are raised.
