# Ember RFCs

The RFC ("request for comments") process is how Ember designs and
achieves consensus on "substantial" proposed changes.

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put
through a bit of a design process and produce a consensus among the Ember
core teams.

The process is intended to provide a consistent and controlled path for new
features and changes to enter the framework. RFCs can be created by any member
of the community.

---
## Quick links for Pull Requests for proposed and advancing RFCs
[Quick links for Pull Requests for proposed and advancing RFCs]: #quick-links-for-pull-requests-for-proposed-and-advancing-rfcs

- [Proposed RFCs](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc+label%3AS-Proposed)
- [Exploring RFCs](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc+label%3AS-Exploring)

- [PRs to advance an RFC to Ready for Release Stage](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc+label%3A%22S-Ready+for+Release%22)
- [PRs to advance an RFC to Released Stage](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc+label%3AS-Released)
- [PRs to advance an RFC to Recommended Stage](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc+label%3AS-Recommended)

- [RFCs in Final Comment Period for any Stage](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc+label%3A%22Final+Comment+Period%22)
- [RFCs in FCP to Close](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc+label%3A%22FCP+to+close%22)


**[RFC Website](https://rfcs.emberjs.com)**

This site shows the current state and text of all RFCs in Accepted or later stages.

---

## Table of Contents
[Table of Contents]: #table-of-contents

- [Introduction](#ember-rfcs)
- [Quick links](#quick-links-for-pull-requests-for-proposed-and-advancing-rfcs)
- [Table of Contents]
- [How it works](#how-it-works)
- [Stages]
  - [Proposed]
  - [Exploring]
  - [Accepted]
  - [Ready for Release]
  - [Released]
  - [Recommended]
- [Following RFC Progress](#following-rfc-progress)
- [Commenting on an RFC](#commenting-on-an-rfc)
- [Creating an RFC](#creating-an-rfc)
  - [When you need to follow this process]
  - [Gathering feedback before submitting an RFC]
  - [The Process]
    - [How to create a new RFC]
    - [Finding a champion]
- [Implementing an RFC]
- [Editing merged RFCs]
- [Reference]
  - [Relevant Teams]
  - [Final Comment Periods (FCP)](#final-comment-periods-fcp)
    - [FCP to close]
- [For Core Team Members](#for-core-team-members)


## How it works
[How it works]: #how-it-works

### New Proposals

New proposals for changes are submitted as pull requests to this repo using one
of the RFC templates, depending on what the RFC is proposing. The newly opened RFC
is in the [Proposed] stage. The RFC is then reviewed and commented upon by
core team members and the community. The core teams may move the RFC to the
[Exploring] stage via a label if they believe the RFC is worth exploring further.
From there, with refinement, community consensus, and agreement by the relevant
core teams, the RFC enters a [Final Comment Period](#final-comment-periods-fcp) (FCP).
If that period is successful, with no new unaddressed concerns raised, the RFC
is merged and is now in the [Accepted] stage.

An RFC that does not achieve consensus to be accepted may enter a
[Final Comment Period](#final-comment-periods-fcp) to move the RFC to the
[Closed] stage.

### Merged Proposals

Once the RFC is accepted, a draft pull request is automatically opened to advance
the RFC to the [Ready for Release] stage. This pull request has a template that outlines
what is necessary to merge the RFC into this stage. This pull request is where the
community can track the progress of the implementation of the RFC. Once the criteria
to move to [Ready for Release] are met, including commitment of the team to the
feature's stability, the PR to advance may be merged after a [Final Comment Period](#final-comment-periods-fcp).
Each core team needs to review the PR and approve it before the end of the FCP.
If the FCP is successful, the RFC is merged and becomes [Ready for Release].

Once the RFC is merged as [Ready for Release], a pull request is automatically opened
to advance the RFC to the [Released] stage. This pull request has a template that outlines
what is necessary to merge the RFC into this stage. The pull request is merged
when those requirements are met and the relevant team has consensus that the RFC
has been [Released]. No FCP is required for this stage.

Once the RFC is merged as [Released], a pull request is automatically opened to
advance the RFC to the [Recommended] stage. This pull request has a template that outlines
what is necessary to merge the RFC into this stage and can be used for community
coordination. This advancement requires a
[Final Comment Period](#final-comment-periods-fcp) to merge.

At this point, the RFC is [Recommended] and complete.

RFCs that have been merged to the repo can also move to the [Discontinued] stage at
any point via a pull request. This is done when the RFC is no longer relevant or
the feature is no longer supported.

Read more about the RFC [Stages].

## Stages
[Stages]: #stages

Here are all the stages that an RFC can be in:

| Stage| Description| Requires FCP to enter? |
| -----| -----------|----- |
| [0 - Proposed](#Proposed) | A proposal for a change to Ember or its processes that is offered for community and team evaluation. | no |
| [1 - Exploring](#Exploring) | An RFC deemed worth pursuing but in need of refinement. | no |
| [2 - Accepted](#Accepted) | A fully specified RFC. Waiting for or in the process of implementation. | yes |
| [3 - Ready for Release](#Ready-for-Release) | The implementation of the RFC is complete, including learning materials. | yes |
| [4 - Released](#Released) | The work is published. If it is codebase-related work, it is in a stable version of the relevant package(s). | no |
| [5 - Recommended](#Recommended) | The feature/resource is recommended for general use. | yes |

There are two additional statuses for RFCs that will not move forward:

- **[Discontinued](#Discontinued)** - a previously [Accepted] RFC that is either in conflict with Ember's evolving programming model or is superseded by another active RFC.
- **[Closed](#Closed)** - Proposed RFCs that will not be moved into the next stage.

### Proposed
[Proposed]: #proposed

Proposed RFCs are opened as pull requests to the RFC repository. Anybody may
create an RFC. The format should follow the templates in the RFC repository.

An RFC's number is the number of it's original proposal PR.

From "Proposed" an RFC may move to [Exploring],
or [Closed] stages.
To move to [Closed] an FCP is required as in the existing process.
A "Proposed" RFC may be moved to "Exploring" by consensus of the relevant
team(s) without an FCP. See [Exploring].

### Exploring
[Exploring]: #exploring

An Exploring RFC is one the Ember team believes should be pursued, but the RFC
may still need some more work, discussion, answers to open questions,
and/or a champion before it can move to the next stage.

An RFC is moved into Exploring with consensus of the relevant teams. The
relevant team expects to spend time helping to refine the proposal. The
RFC remains a PR and will have an `Exploring` label applied.

An Exploring RFC that is successfully completed can move to [Accepted]
with an FCP is required as in the existing process. It may also be moved to
[Closed] with an FCP.

### Accepted
[Accepted]: #accepted

An RFC that has been "accepted" has complete prose and has successfully passed
through an "FCP to Accept" period in which the community has weighed in and
consensus has been achieved on the direction. The relevant teams believe that
the proposal is well-specified and ready for implementation. The RFC has a
champion within one of the relevant teams.

If there are unanswered questions, we have outlined them and expect that they
will be answered before [Ready for Release].

When an RFC is merged and moved to "Accepted", a new PR will be opened to move
it to [Ready for Release]. This PR should be used to track
the implementation progress and gain consensus to move to the next stage.

### Ready for Release
[Ready for Release]: #ready-for-release

The implementation is complete according to plan outlined in the RFC, and is in
harmony with any changes in Ember that have occurred since the RFC was first
written. This includes any necessary learning materials. At this stage, features
or deprecations may be available for use behind a feature flag, or with an
optional package, etc. The team reviews the work to determine when it can be
included in a stable release. For codebase changes, there are no open questions
that are anticipated to require breaking changes; the Ember team is ready to
commit to the stability of any interfaces exposed by the current implementation
of the feature. Today, this would be the "go/no-go" decision by a particular
team.

This stage should include a list of criteria for determining when the proposal
can be considered [Recommended] after being [Released].

A PR is opened on the repo (see [Accepted]) to move an accepted RFC
into this stage. An FCP is required to move into this stage.

Each Ember core team will be requested as a reviewer on the PR to move into this
stage. A representative of each team adds a review. If a team does not respond
to the request, and after the conclusion of the FCP, it is assumed that the
release may proceed.

### Released
[Released]: #released

The work is published. If it is codebase-related work, it is in a stable version
of the relevant package(s). If there are any critical deviations from the
original RFC, they are briefly noted at the top of the RFC.

If the work for an RFC is spread across multiple releases of Ember or other packages,
the RFC is considered to be in the Released stage when all features are available in
stable releases and those packages and versions are noted in the RFC frontmatter.

Ember's RFC process can be used for process and work plans that are not about code.
Some examples include Roadmap RFCs, changes to the RFC process itself, and
changes to learning resources.
When such an RFC is a candidate for Released, the work should be shipped as described,
and the result should presented to the team with the intent of gathering feedback
about whether anything is missing. If there is agreement that the work is complete,
the RFC may be marked "Released" and a date is provided instead of a version.

An RFC is moved into "Released" when the above is verified by consensus of the
relevant team(s) via a PR to update the stage.

### Recommended
[Recommended]: #recommended

The "Recommended" stage is the final milestone for an RFC. It provides a signal
to the wider community to indicate that a feature has been put through its
ecosystem paces and is ready to use.

The "Recommended" stage is most important for suites of features that are designed
as a number of separate RFCs. It allows the Ember maintainers to stabilize individual
features once they are technically feature complete, an important goal for maintaining
technical velocity.

To reach the "Recommended" stage, the following should be true:

- If appropriate, the feature is integrated into the tutorial and the guides prose.
  API documentation is polished and updates are carried through to other areas of
  API docs that may not directly pertain to the feature.
- If the proposal replaces an existing feature, the addon ecosystem has largely
  updated to work with both old and new features.
- If the proposal updates or replaces an existing feature, high-quality codemods are
  available
- If needed, Ember debugging tools as well as popular IDE support have been
  updated to support the feature.
- If the feature is part of a suite of features that were designed to work together
  for best ergonomics, the other features are also ready to be "Recommended".
- Any criteria for "Recommended" for this proposal that were established in the
  [Ready For Release] stage have been met.

An RFC is moved into "Recommended" via PR to update the stage. An FCP is required
to enter this stage. Multiple RFCs may be moved as a batch into "Recommended" with
the same PR.

### Closed
[Closed]: #closed

A [Proposed] or [Exploring] RFC may be closed after an FCP period. This is the same
as the existing process. A closed RFC is discontinued.

### Discontinued
[Discontinued]: #discontinued

A previously [Accepted] RFC may be discontinued at any point. The RFC
may be superseded, out-of-date, or no longer consistent with the direction of
Ember.

## Following RFC Progress
[Following RFC Progress]: #following-rfc-progress

Following the RFC repository is the best way to keep up with the proposed
changes to Ember and with the implementation of accepted changes.

Watching the repository will alert you to any newly created RFCs. Setting up
notifications on a particular pull request will help you follow the progress of
that RFC.

RFCs in [Final Comment Periods](#final-comment-periods-fcp) are labeled with
[Final Comment Period](https://github.com/emberjs/rfcs/pulls?q=is%3Apr+is%3Aopen+sort%3Aupdated-desc+label%3A%22Final+Comment+Period%22)
and are announced in `#news-and-announcements` on [Ember Discord](https://discord.gg/emberjs) and
on the [Ember twitter account](https://twitter.com/emberjs).

[Quick links](#quick-links-for-pull-requests-for-proposed-and-advancing-rfcs)
are provided at the top of the README to help you review what you are interested in.


## Commenting on an RFC
[Commenting on an RFC]: #commenting-on-an-rfc

Comments are the very heart of the RFC process. Comments are the primary way community
members and core team members can provide feedback and collaborate on the design
of a feature.

Comments should also be used to ensure the author has completed the RFC template
and addressed implications of the feature across the Ember project, including
documentation, addons, and tooling.

Comments should be constructive and respectful, following our
[Community Guidelines](http://emberjs.com/guidelines/).

## Creating an RFC
[Creating an RFC]: #creating-an-rfc

### When you need to follow this process
[When you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make "substantial"
changes to Ember, Ember Data, Ember CLI, their documentation, or any other
projects under the purview of the [Ember core teams](https://emberjs.com/team/).
What constitutes a "substantial" change is evolving based on community norms,
but may include the following:

   - A new feature that creates new API surface area, and would
     require a [feature flag] if introduced.
   - The removal of features that already shipped as part of the release
     channel.
   - The introduction of new idiomatic usage or conventions, even if they
     do not include code changes to Ember itself.

Some changes do not require an RFC:

   - Rephrasing, reorganizing or refactoring
   - Addition or removal of warnings
   - Additions that strictly improve objective, numerical quality
criteria (speedup, better browser support)
   - Additions only likely to be _noticed by_ other implementors-of-Ember,
invisible to users-of-Ember.

If you submit a pull request to implement a new feature without going
through the RFC process, it may be closed with a polite request to
submit an RFC first.

### Gathering feedback before submitting an RFC
[Gathering feedback before submitting an RFC]: #gathering-feedback-before-submitting-an-rfc

It's often helpful to get feedback on your concept before diving into the
level of API design detail required for an RFC. **You may open an
issue on this repo to start a high-level discussion**, with the goal of
eventually formulating an RFC pull request with the specific implementation
 design. We also **_highly recommend_** sharing drafts of RFCs in `#dev-rfc` on
the [Ember Discord](https://discord.gg/emberjs) for early feedback.

### The Process
[The Process]: #the-process

The process for a successful RFC involves several [Stages] that take
the proposed change all the way from proposal to implemented and widely adopted.
A newly opened RFC is in the first stage, appropriately named, [Proposed].

#### How to create a new RFC
[How to create a new RFC]: #how-to-create-a-new-rfc

* Fork the RFC repo http://github.com/emberjs/rfcs
* Copy the appropriate template. For most RFCs, this is `0000-template.md`,
for deprecation RFCs it is `deprecation-template.md`.
Copy the template file to `text/0000-my-feature.md`, where
'my-feature' is descriptive. Don't assign an RFC number yet.
* Fill in the RFC. Put care into the details: **RFCs that do not
present convincing motivation, demonstrate understanding of the
impact of the design, or are disingenuous about the drawbacks or
alternatives tend to be poorly-received**.
* Fill in the relevant core teams. Use the [table below](#relevant-teams) to map from projects to
teams.
* Submit a pull request. As a pull request the RFC will receive design
feedback from the larger community, and the author should be prepared
to revise it in response. The RFC is now in the [Proposed] stage.
* Find a champion on the relevant core team. The champion is responsible for
shepherding the RFC through the RFC process and representing it in core team
meetings.
* Update the pull request to add the number of the PR to the filename and
add a link to the PR in the header of the RFC.
* Build consensus and integrate feedback. RFCs that have broad support
are much more likely to make progress than those that don't receive any
comments.
* From here, the RFC moves to the [Exploring] stage or [Closed] in the process
explained in [Stages].

### Finding a champion
[Finding a Champion]: #finding-a-champion

The RFC Process requires finding a champion from the relevant core teams. The
champion is responsible for representing the RFC in team meetings, and for
shepherding its progress. [Read more about the Champion's job](#champion-responsibilities)

- *Find one via Discord.*
  The `dev-rfc` channel on the [Ember Discord](https://discord.gg/emberjs) is
  reserved for the discussion of RFCs. We highly recommend circulating early drafts of
  your RFC in this channel to both receive early feedback and to find a champion.

- *Ask for a champion via an issue, or in the RFC itself.*
  We monitor the RFC repository. We will circulate requests for champions, but highly
  recommend discussing the RFC in Discord.

## Implementing an RFC
[Implementing an RFC]: #implementing-an-rfc

Coordination and tracking of implementation of an RFC is done primarily on the
pull request to advance the RFC to the [Ready for Release] stage.

The author of an RFC is not obligated to implement it. Of course, the
RFC author (like any other developer) is welcome to work on the implementation.

If you are interested in working on the implementation for an [Accepted]
RFC, but cannot determine if someone else is already working on it,
please ask (e.g. by leaving a comment on the pull request to advance the RFC to
the [Ready for Release] stage).

### Editing merged RFCs
[Editing merged RFCs]: #editing-merged-rfcs

A merged RFC may be edited via a Pull Request process. Edits may include things like:

- Updating the stage
- An optional note at the top that summarizes minor adjustments to the RFC
design, at the time that the RFC's work became available for general use. This
note can be very brief, and link out to other resources like a blog post. For
example, an update might simply say "See `<blog post>` for more information
about this feature." This note is not intended to be updated across time.
- Updating any part of the RFC prose, in order to keep a written record of the
changes and rationale.

Major changes should have a new RFC. The old RFC is moved to the [Discontinued]
stage when the replacement is merged.

## Reference
[Reference]: #reference

### Relevant Teams
[Relevant Teams]: #relevant-teams

The RFC template requires indicating the relevant core teams. The following table
offers a reference of teams responsible for each project. Please reach out for
further guidance.

|   Core Team   | Project/Topics                               | Github Team                |
|---------------|----------------------------------------------|----------------------------|
| Ember.js      | Ember.js                                     | `@emberjs/Framework`       |
| Ember Data    | Ember Data                                   | `@emberjs/ember-data-core` |
| Ember CLI     | Ember CLI                                    | `@emberjs/cli`             |
| Learning      | Documentation, Website, learning experiences | `@emberjs/learning-core`   |
| TypeScript    | TypeScript integration and design            | `@emberjs/typescript-core` |
| Steering      | Governance                                   | `@emberjs/steering`        |

## Final comment periods (FCP)
[FCP]: #final-comment-periods-fcp

For certain stage advancements, a _final comment period_ (FCP) is required. This
is a period lasting 7 days. The beginning of this period will be signaled with a
comment and tag on the RFC's pull request. Furthermore, a message will be posted in
`#news-and-announcements` on [Ember Discord](https://discord.gg/emberjs) and on the
[official Ember Twitter account](https://twitter.com/emberjs) to attract the community's attention.

An RFC can be modified based upon feedback from the [core teams] and community
during the final comment period. Significant modifications may trigger a new
final comment period.

At the end of a successful FCP, the RFC moves into the stage specified.

### FCP to close
[FCP to close]: #fcp-to-close

An RFC may be closed or discontinued by the [core teams] after public discussion
has settled and comments have been made summarizing the rationale for closing.
The RFC will enter a "final comment period to close" lasting 7 days. At the end
of the "FCP to close" period, the PR will be closed.

An RFC author may withdraw their own RFC by closing it themselves.

## For Core Team Members
[For Core Team Members]: #for-core-team-members

### Reviewing RFCs

Each core team is responsible for reviewing open RFCs. The team must ensure
that if an RFC is relevant to their team's responsibilities the team is
correctly specified in the 'Relevant Team(s)' section of the RFC front-matter.
The team must also ensure that each RFC addresses any consequences, changes, or
work required in the team's area of responsibility.

As it is with the wider community, the RFC process is the time for
teams and team members to push back on, encourage, refine, or otherwise comment
on proposals.

### Reviewing entry into [Ready for Release] Stage

As described in [Ready for Release], each core team is responsible for
reviewing RFCs that are ready to move to that stage.

### Referencing RFCs

- When mentioning RFCs that have been merged, link to the merged version,
not to the pull-request.

### Champion Responsibilities

* Achieving consensus from the team(s) to move the RFC through the stages of
the RFC process.
* Ensuring the RFC follows the RFC process.
* Shepherding the planning and implementation of the RFC. Before the RFC is
[Accepted], the champion may remove themselves. The champion may find a replacement
champion at any time.

### Helpful checklists for Champions

#### Becoming champion of an RFC
- [ ] Assign the RFC to yourself

#### Advancing the RFC to the next stage
- [ ] Achieve consensus to move to the next stage from relevant core teams
- [ ] If the stage requires an FCP to enter, comment in the RFC to address any
  outstanding issues and to proclaim the start of the FCP period, and tweet
  from `@emberjs` about the FCP
- [ ] Ensure the RFC has had the filename and header updated with the PR number
- [ ] After the FCP period (if any), merge the RFC PR
- [ ] Ensure relevant teams plan out what is necessary to implement in the next
  stage
- [ ] Prepare for advancing the RFC to the next stage if applicable (e.g.
  opening a placeholder PR)

For RFCs opened before the implementation of
[Staged RFCs](https://github.com/emberjs/rfcs/blob/master/text/0617-rfc-stages.md)
did not get advancement PRs to their next stage. See the
[automations docs](https://github.com/emberjs/rfcs/blob/master/.github/READTHIS.md#trigger-opening-advancement-pryml) for instructions on how to trigger the advancement workflow.

#### Move to FCP to Close
- [ ] Achieve consensus to move to "FCP to Close" from relevant core teams
- [ ] Comment in the RFC to explain the decision

#### Closing an RFC
- [ ] Comment about the end of the FCP period with no new info
- [ ] Close the PR

#### Running the app locally

The app that renders our RFCs, [rfcs.emberjs.com/](https://rfcs.emberjs.com/)
can be found in [ember-learn/rfcs-app](https://github.com/ember-learn/rfcs-app).
The app deploys automatically via GitHub hooks.

**Ember's RFC process owes its inspiration to the [Rust RFC process]**

[Rust RFC process]: https://github.com/rust-lang/rfcs
[core teams]: http://emberjs.com/team/
[feature flag]: http://emberjs.com/guides/contributing/adding-new-features/
[core team notes]: https://github.com/emberjs/core-notes/tree/master/ember.js
