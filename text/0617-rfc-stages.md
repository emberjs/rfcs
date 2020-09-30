- Start Date: 2020-04-22
- Relevant Team(s): (fill this in with the team(s) to which this RFC applies)
- RFC PR: https://github.com/emberjs/rfcs/pull/617
- Tracking: (leave this empty)

# RFC Stages

## Summary

Ember's users should be able to look at an RFC and know more about how close it is to being part of a stable release. This proposal introduces explicit stages for RFCs, covering the steps from the initial draft to the shipped result. Inspired by TC39, these stages are a communication and collaboration tool. They can give the community greater visibility into Ember's development, encourage participation, and improve cross-team coordination. This RFC does not aim to substantially change the existing process, but rather apply labels to what already happens informally.

## Motivation

It can be difficult for both users and contributors of Ember to know the status of a new feature. Has it been approved? Is it a work in progress? What still needs to be done? Which version of Ember did it ship in?

We can see that stages are needed when we consider the following:

1. Community feedback that it was difficult to follow along with the progression of Ember Octane's features
2. Challenges uncovered in the existing RFC Tracking process
3. The long-standing, but stagnant goal of providing a public-facing status board
4. Confusion about what it means for an RFC to be merged. There is a common myth that a merged RFC indicates that a feature will be in the next release, when instead it is a green flag for implementation work to begin.

To understand the problem, it is helpful to think of the informal stages a merged RFC already can progress through: We have RFCs for which we have needed to indicate a change in direction ([MU](https://emberjs.github.io/rfcs/0143-module-unification.html)), RFCs that have needed to be clarified/replaced ([Named Blocks](https://emberjs.github.io/rfcs/0226-named-blocks.html)), RFCs waiting to be implemented, and RFCs whose features have been included in a stable release. Each of these scenarios was handled in a thoughtful manner, but they were difficult for users to follow along with.

This proposal aims to build on the success of RFC 300, [RFC (Request for Comments) Process Update]([https://github.com/emberjs/rfcs/blob/master/text/0300-rfc-process-update.md](https://github.com/emberjs/rfcs/blob/master/text/0300-rfc-process-update.md)).

## Detailed design

### Current Process

For details of the current RFC process, see the current [README of the RFCs repo](https://github.com/emberjs/rfcs/blob/2a7b9e8605a56277a4dd515bf8e192d9ac09c12f/README.md):

### Stages

A successful RFC can move through the following sequential stages. Whenever an RFC moves to the next stage, there is a PR to update it.

| Stage| Description| Requires FCP to enter? |
| -----| -----------|----- |
| [0 - Proposed](#Proposed) | A proposal for a change to Ember or its processes that is offered for community and team evaluation. | no |
| [1 - Exploring](#Exploring) | An RFC deemed worth pursuing but in need of refinement. | no |
| [2 - Accepted](#Accepted) | A fully specified RFC. Waiting for or in the process of implementation. | yes |
| [3 - Ready for Release](#Ready-for-Release) | The implementation of the RFC is complete, including learning materials. | yes |
| [4 - Released](#Released) | The work is published. If it is codebase-related work, it is in a stable version of the relevant package(s). | no |
| [5 - Recommended](#Recommended) | The feature/resource is recommended for general use. | yes |

There are two additional statuses for RFCs that will not move forward:

- **[Discontinued](#Discontinued)** - a previously Accepted RFC that is either in conflict with Ember's evolving programming model or is superseded by another active RFC.
- **[Closed](#Closed)** - Proposed RFCs that will not be moved into the next stage.

#### Proposed

Proposed RFCs are opened as pull requests to the RFC repository. Anybody may 
create an RFC. The format should follow the templates in the RFC repository. 
There is currently a default template and a deprecation RFC template. This 
process is discussed in depth in the RFCs repo README.

An RFC's number is the number of it's original proposal PR.

From "Proposed" an RFC may move to [Exploring](#Exploring),
or [Closed](#Closed) stages.
To move to [Closed](#Closed) an FCP is required as in the existing process.
A "Proposed" RFC may be moved to "Exploring" by consensus of the relevant
team(s) without an FCP. See [Exploring](#Exploring).

#### Exploring

An Exploring RFC is one the Ember team believes should be pursued, but the RFC 
may still need some more work, discussion, answers to open questions, 
and/or a champion before it can move to the next stage. 

An RFC is moved into Exploring with consensus of the relevant teams. The 
relevant team expects to spend time helping to refine the proposal. The
RFC remains a PR and will have an `Exploring` label applied.

An Exploring RFC that is successfully completed can move to [Accepted](#Accepted) 
with an FCP is required as in the existing process. It may also be moved to
[Closed] with an FCP.

#### Accepted

An RFC that has been "accepted" has complete prose and has successfully passed
through an "FCP to Accept" period in which the community has weighed in and consensus 
has been achieved on the direction. The relevant teams believe that the 
proposal is well-specified and ready for implementation. 
The RFC has a champion within one of the relevant teams. 

This is equivalent to today's RFCs being merged.

If there are unanswered questions, we have outlined them and expect that they 
will be answered before [Ready for Release](#Ready-for-Release).
 
When an RFC is merged and moved to "Accepted", a new PR will be opened to move 
it to [Ready for Release](#Ready-for-Release). This PR should be used to track 
the implementation progress and gain consensus to move to that next stage.

#### Ready for Release

The implementation is complete according to plan outlined in the RFC, 
and is in harmony with any changes in Ember that have occurred since the RFC was first written.
This includes any necessary learning materials.
At this stage, features or deprecations may be available for use behind a feature flag,
or with an optional package, etc. 
The team reviews the work to determine when it can be included in a stable release.
For codebase changes, there are no open questions that are anticipated to require
breaking changes; the Ember team is ready to commit to the stability of any 
interfaces exposed by the current implementation of the feature.
Today, this would be the "go/no-go" decision by a particular team. 

This stage should include a list of criteria for determining when the proposal 
can be considered [Recommended](#Recommended) after being [Released](#Released). 

A PR is opened on the repo (see [Accepted](#Accepted)) to move an accepted RFC 
into this stage. An FCP is required to move into this stage.

Each Ember core team will be requested as a reviewer on the PR to move into this stage. 
A representative of each team adds a review. If a team does not respond to the 
request, and after the conclusion of the FCP, it is assumed that the release may proceed.

#### Released

The work is published. If it is codebase-related work, it is in a stable version 
of the relevant package(s). If there are any critical deviations from the original RFC, 
they are briefly noted at the top of the RFC.

If the work for an RFC is spread across multiple releases of Ember or other packages, 
the RFC is considered to be in the Released stage when all features are available in
stable releases and those packages and versions are noted in the RFC frontmatter.

Ember's RFC process can be used for process and work plans that are not about code. 
Some examples include Roadmap RFCs, this RFC itself, and changes to learning resources.
When such an RFC is a candidate for Released, the work should be shipped as described, 
and the result should presented to the team with the intent of gathering feedback 
about whether anything is missing. If there is agreement that the work is complete, 
the RFC may be marked "Released" and a date is provided instead of a version.

An RFC is moved into "Released" when the above is verified by consensus of the 
relevant team(s) via PR to update the stage.

#### Recommended

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
[Ready For Release](#Ready-for-Release) stage have been met.

An RFC is moved into "Recommended" via PR to update the stage. An FCP is required
to enter this stage. Multiple RFCs may be moved as a batch into "Recommended" with
the same PR.

#### Closed

A [Proposed](#Proposed) or [Exploring](#Exploring) RFC may be closed after an FCP period. This is the same 
as the existing process. A closed RFC is discontinued.

#### Discontinued

A previously [Accepted](#Accepted) RFC may be discontinued at any point. The RFC
may be superseded, out-of-date, or no longer consistent with the direction of 
Ember. 


### Editing merged RFCs

A merged RFC may be edited via a Pull Request process. Edits may include things like:

- Updating the stage
- An optional note at the top that summarizes minor adjustments to the RFC design, at the time that the RFC's work became available for general use. This note can be very brief, and link out to other resources like a blog post. For example, an update might simply say "See `<blog post>` for more information about this feature." This note is not intended to be updated across time.
- Updating any part of the RFC prose, in order to keep a written record of the changes and rationale.

Today, these types of changes do happen. We just don't write them down. So, merge criteria should be very loose.

Major changes should have a new RFC. The old RFC is later marked "Discontinued" when the replacement is merged. 

### Changes to RFC meta

RFC meta ("frontmatter") is the block of text at the start of an RFC that includes data about its stage, links to relevant info, etc.

Before:

    - Start Date: (fill me in with today's date, YYYY-MM-DD)
    - Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
    - RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
    - Tracking: (leave this empty)

After:

    - Stage: Proposed (later updated to other stages)
    - Start Date: (fill me in with today's date, YYYY-MM-DD)
    - Release date: Unreleased (later update with YYYY-MM-DD)
    - Release Versions: (Include any package with work necessary for the feature, n/a for non-code RFCs)
      - ember-source: vX.Y.Z
      - ember-data: vX.Y.Z 
    - Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
    - RFC PR: (after opening the Propsoal RFC PR, update this with a link to it and update the file name)
     
For RFCs that have moved through to at least the [Accepted](#Accepted) stage, 
this data will be used to add and update a block at the top of the RFC prose to
indicate the current stage, with a brief explanation of that stage. It should 
link to any open PRs to update the stage of the RFC. 

We will make use of automation to add/update a section to the RFC with links to 
each PR that caused the RFC to move to a new stage.

`Tracking` will be removed for new RFCs. Links that would have appeared here should
be found on the PR to move the RFC to [Ready for Release](#Ready-for-Release).

### Reconciling past RFCs

For codebase-related RFCs that have already been merged, the release version is only required to be added to RFCs whose implementation was released after February 1st, 2018 (near the release of Ember 3). This is to preserve the time and effort of our contributors and volunteers.

A stage will be applied to all previously merged RFCs.

### Non-code RFCs

The names of the stages make the most sense with RFCs that propose features in 
Ember, with code that will follow a release process. For many non-code RFCs, 
such as this one, those names, especially of later stages, may seem "off". 
However, it is still  valuable to have a stage for every RFC where the teams and 
community agree that the RFC has been implemented ([Ready for Release](#Ready-for-Release)) and a 
stage where the community agree that the RFC has been polished ([Recommended](#Recommended)).
We may further refine this in a future RFC as we learn more. 

## How we teach this

- The Stages section above will be added to the README of the RFCs repository.
- Frontmatter will be added to the template for new RFCs (see below)
- A blog post will announce the updates to the RFC process
- Past RFCs should be updated to include their stage and release version, following the description in "Reconciling past RFCs."
- We will use automation to add helpful links and guidance to the RFC files themselves.

### Stages as communication tool

Additionally, this RFC would unlock our ability to create a status board where everyone can easily see the progress of RFC. Today, if you need to know when an RFC feature was available in a stable release, you need to comb through release blog posts. Instead, we could use the frontmatter as data that powers the app. This app is not a requirement for this RFC, but rather an example of how we could use the information provided by stages.

### Non-goals

This RFC does not intend to:

- substantially alter the functional process for RFCs
- speed up the RFC process
- change the timing of when a feature is released
- exhaustively cover all steps and processes

## Case study

Here is how we could have applied this model to Tracked Properties, which was split across two RFCs: [#410](https://github.com/emberjs/rfcs/blob/master/text/0410-tracked-properties.md) and [#478](https://github.com/emberjs/rfcs/blob/master/text/0478-tracked-properties-updates.md).

### 0 - Proposed

A PR is opened to the RFCs repo for Tracked Properties. The framework team talks 
about the RFC in the weekly meetings, and there's general agreement to pursue the 
idea. The RFC moves to [Exploring](#Exploring). A label is added to the PR to 
indicate that stage.

### 1 - Exploring

@pzuraq keeps adding details to the RFC, explores the design space, and 
collaborates with others to get to the final design. The full story has been 
thought out. @pzuraq expects to have the time resources to work on implementation. 
There were no "known unknowns" questions. The RFC reaches consensus. The RFC 
makes it through a week-long FCP process. The PR is merged and the stage 
is now [Accepted](#Accepted). 

### 2 - Accepted

@pzuraq works on implementation. The feature is enabled in canary under feature flag. 
We learn that the feature causes a behavior regression in the interop story. 
@pzuraq works out a new plan to accommodate interop, then opens a PR to update the RFC prose.

The implementation is complete. There are API docs. A PR is opened to move the proposal to 
[Ready for Release](#Ready-for-Release). It includes a list of criteria required 
for this feature to be [Recommended](#Recommended).
On the PR to move to [Ready for Release], each Ember team is requested as a reviewer. 
Each team reviews the RFC and implementation, ensuring that any changes to the 
projects they are responsible for have been completed and that the criteria for
[Recommended](#Recommended) also considers those areas. After a successful FCP 
period, the PR is merged and the stage is now [Ready for Release](#Ready-for-Release).

### 3 - Ready for Release

The mechanics of releasing the feature proceed. The feature proceeds through 
Ember.js' beta cycle. The next stable of Ember.js is released with the feature
available. The API docs are published. A PR to update the stage to [Released](#Released)
and the frontmater with release details is opened and merged with the consensus 
of the framework team.

### 4 - Released

The feature is available for use by users of Ember.js. The learning team works to
carry through the concepts of 'Tracked Properties' to the tutorial and guides. Changes
to API doc examples are prepared. Other criteria for moving to Recommended are worked on, as defined
in the [Ready for Release](#Ready-for-Release) step. This work is documented on
a PR to move the proposal to [Recommended](#Recommended). This proposal, along with
several others, are PRed to move to Recommended at the same time, 
as part of Octane around Ember.js 3.14. It is determined that the features are not 
yet polished enough and criteria to get to Recommended has not yet
been met. More work proceeds and the features are again proposed as Recommended
and put into a "FCP for Recommended", it succeeds and the stages of several 
proposals are updated to Recommended as part of the Octane Edition.

### 5 - Recommended

The feature is released and suggested for use by the wider Ember community. They
should encounter a polished feature that has ecosystem support. The feature should
be well represented in learning materials and the guides, tutorial and API docs 
use the feature in a consistent manner. 

How was the actual process different from the imaginary case study above? In reality, there were two separate RFCs needed to land the feature, and there were fewer opportunities for people to follow along, give input, and understand the status.

## Drawbacks

Making updates to past RFCs can generate a lot of  GitHub notifications for people who watch the repository. We think that the benefit of knowing when an RFC is available in a stable release outweighs this drawback. Additionally, in the future, if an app displays the frontmatter data, someone could use that as their main source of information. Lastly, the Ember Times does a great job curating RFC updates, so someone could watch it instead if the RFC repo itself has too much information.

## Alternatives/State of the Art

This section covers how other projects have handled this type of tracking, and how we have done it in the past.

### Vue

[Vue's stages](https://github.com/vuejs/rfcs#the-rfc-life-cycle) are concise and to the point. This is worth considering because you don't need to know the process deeply in order to understand it. The lower number of stages equates to less overhead, but also less fidelity.

- **Pending:** when the RFC is submitted as a PR.
- **Active:** when an RFC PR is merged and undergoing implementation.
- **Landed:** when an RFC's proposed changes are shipped in an actual release.
- **Rejected:** when an RFC PR is closed without being merged.

### TC39

TC39 is an interdisciplinary group that is responsible for determining the direction of JavaScript. Staged proposals are greatly inspired by their process and the success. The shorthand provided by stage names helps bring clarity in communication.

 [Their process](https://tc39.es/process-document/) is divided into five numbered stages:

0. Strawperson

1. Proposal
2. Draft
3. Candidate
4. Finished

This format is not exactly suited to Ember because stages 0-3 are what we already do. What we need to offer is more detail for stages 3-4. However, thanks to TC39, we can see how useful it would be to have clarity and a shared language around the status of a feature.

### Rust

The [Rust RFC lifecycle](https://github.com/rust-lang/rfcs#the-rfc-life-cycle) is very similar to Ember's, because ours was inspired by it. Following an FCP (Final Comment Period), an RFC is either merged, closed, or postponed. Merged RFCs are considered to be "active."

The concept of postponed RFCs is relevant to Ember's development, and possibly a good lesson to take:

- Some RFC pull requests are tagged with the "postponed" label when they are
closed (as part of the rejection process). An RFC closed with "postponed" is
marked as such because we want neither to think about evaluating the proposal
nor about implementing the described feature until some time in the future, and
we believe that we can afford to wait until then to do so. Historically,
"postponed" was used to postpone features until after 1.0. Postponed pull
requests may be re-opened when the time is right. We don't have any formal
process for that, you should ask members of the relevant sub-team.
- Usually an RFC pull request marked as "postponed" has already passed an
informal first round of evaluation, namely the round of "do we think we would
ever possibly consider making this change, as outlined in the RFC pull request,
or some semi-obvious variation of it." (When the answer to the latter question
is "no", then the appropriate response is to close the RFC, not postpone it.)

 As of the writing of this RFC, there are over [70 open pull requests](https://github.com/emberjs/rfcs/pulls) for Ember's RFCs. Having so many open PRs is confusing for everyone, yet simply closing many of these PRs would not reflect the reality of the wish to bring these good ideas into Ember.

Another aspect of Rust's process that is relevant is the idea of modification, which is already part of Ember's process, but not widely used:

- Modifications to "active" RFCs can be done in follow-up pull requests. We strive to write each RFC in a manner that it will reflect the final design of the feature; but the nature of the process means that we cannot expect every merged RFC to actually reflect what the end result will be at the time of the next major release.

Many of Ember's merged RFCs are not implemented exactly as written. Once implementation is underway, new knowledge gained influences design. In other cases, implementation takes so long that some aspects of an RFC no longer apply. These decisions should be captured in writing as part of the process.

### Yarn and React

Both Yarn and React have adopted an RFC process similar to Ember and Rust.

You can see an example of updating a merged RFC in this [React RFC Pull Request](https://github.com/reactjs/rfcs/pull/52/files).

### Make no changes to existing process

Following RFC 300, the [RFC tracking issue repository](https://github.com/emberjs/rfc-tracking/issues?q=is%3Aopen+is%3Aissue) was created. This repository contained checklists that were meant to show the general, high-level requirements to ship an RFC in a stable release. Although the issues were helpful to learning resource writers, including the Learning Team, the Ember Times, and contributors, the tracking system did not provide much utility for our broader user community. Tracking issues were rarely updated.

Although it is still helpful to have a detailed view into the work being done, our community's greatest need is to show the overall status of an RFC's implementation in a consistent, reliable way. For this reason, it is necessary that we adopt RFC stages, and not stop at our current process.

## Unresolved questions

There are some ambiguities because RFCs take many forms. Our process cannot cover 100% of scenarios, but we should strive to find answers that cover the vast majority of RFCs.

## Glossary

- **FCP**: "final comment period" An FCP is an opportunity for the community and 
core team members to weight in before an RFC moves a following stage.
