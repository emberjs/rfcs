- Start Date: (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# RFC Stages

## Summary

Ember's users should be able to look at an RFC and know more about how close it is to being part of a stable release. This proposal introduces explicit stages for RFCs, covering the steps from the initial draft to the shipped result. Inspired by TC39, these stages are a communication and collaboration tool tool. They can give the community greater visibility into Ember's development, encourage participation, and improve cross-team coordination. This RFC does not aim to substantially change the existing process, but rather apply labels to what already happens informally.

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

### Stages:

A successful RFC can move through the following stages:

| Stage| Description| Criteria to advance to next step |
| -----| -----------|----------------------------------| 
| Proposed | A pull request made to the RFC repository which is open for public comment and core team evaluation. | "The RFC has been in Final Comment Period, and the relevant core team has consensus to merge it, thus accepting the proposal." |
| Accepted | "The work to integrate the proposal into Ember's codebases or materials may begin. An ""Accepted"" RFC is not necessary in-progress." | "The technical implementation is complete according to plan outlined in the RFC, and is in harmony with any changes in Ember that have occurred since the RFC was first written." |
| Prepared for Release | "The relevant core team reviews the work to determine when it can be included in a stable release. At this stage, features or deprecations may be available for use behind a feature flag, in beta, etc. This is also a time to queue up learning materials, for example." | "The relevant core teams coordinate to include the work in a stable release, and/or publish the materials. If there are any truly critical deviations from the original RFC, they are briefly noted at the top of the RFC." |
| Stable | "The work is in a stable release/is published. If it is codebase-related work, it adheres to SemVer from this point onward." | "If it is a codebase change, it is well documented and has clear migration paths. It is consistent with Ember's mental models." |
| Complete | The feature/resource is recommended for general use. | n/a |

There are two additional statuses for RFCs that will not move forward:

- **Discontinued** - a  previously Accepted RFC that is either in conflict with Ember's evolving programming model or is superseded by another active RFC.
- **Closed** - Proposed RFCs that will not be moved into the next stage.

### Editing merged RFCs

A merged RFC may be edited via a Pull Request process. Edits may include things like:

- Updating the stage
- An optional note at the top that summarizes minor adjustments to the RFC design, at the time that the RFC's work became available for general use. This note can be very brief, and link out to other resources like a blog post. For example, an update might simply say "See `<blog post>` for more information about this feature." This note is not intended to be updated across time.

These edits are reviewed and merged by the relevant core team. Today, these types of changes do happen. We just don't write them down. So, merge criteria should be very loose.

Major changes should have a new RFC. The old RFC is later marked "Discontinued" when the replacement is merged. 

### How this applies to non-code RFCs

Ember's RFC process can be used for process and work plans that are not about code. Some examples include Roadmap RFCs, this RFC itself, and changes to learning resources.

When such an RFC is a Candidate for Release, the work should be shipped as described, and the result should presented to the teams with the intent of gathering feedback about whether anything is missing. If there is agreement that the work is complete, the RFC may be marked "Released" and a date is provided instead of a version.

### Changes to RFC meta

RFC meta is the block of text at the start of an RFC that includes data about its stage, links to relevant info, etc.

Before:

    - Start Date: (fill me in with today's date, YYYY-MM-DD)
    - Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
    - RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
    - Tracking: (leave this empty)

After:

    - Stage: Proposed (later updated to other stages)
    - Release version/date: Unreleased (later updated with vX.Y.Z or YYYY-MM-DD)
    - Start Date: (fill me in with today's date, YYYY-MM-DD)
    - Relevant Team(s): (fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies)
    - RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
    - Tracking: (URLs to project boards, quest issues, etc. Separate by a comma and space.)

### Reconciling past RFCs

For codebase-related RFCs that have already been merged, the release version is only required to be added to RFCs whose implementation was released after February 1st, 2018 (near the release of Ember 3). This is to preserve the time and effort of our contributors and volunteers.

A stage will be applied to all previously merged RFCs.

### Core team sign-off for Release

The Core Teams' review of readiness for release has historically been an informal process. This RFC formalizes the responsibility by creating a stage for gathering confirmation that all necessary work is complete.

When the work described in an RFC is ready for Release, a PR to make the change to the RFC is added by the RFC champion, and each team is requested as a reviewer. A representative of each team adds a review, and the community is also invited to comment via one week of FCP (final comment period). If a team does not respond to the request, it is assumed that the release may proceed. 

Approvals for RFCs whose work is spread across multiple releases and packages are handled on a case-by-case basis. If the work for an RFC is spread across multiple releases of Ember, the RFC is considered to be in the Released stage when all features are complete.

## How we teach this

- The Stages section above will be added to the README of the RFCs repository.
- Frontmatter will be added to the template for new RFCs (see below)
- A blog post will announce the updates to the RFC process
- Past RFCs should be updated to include their stage and release version, following the description in "Reconciling past RFCs."

### Stages as communication tool

Additionally, this RFC would unlock our ability to create a status board where everyone can easily see the progress of RFC. Today, if you need to know when an RFC feature was available in a stable release, you need to comb through release blog posts. Instead, we could use the frontmatter as data that powers the app. This app is not a requirement for this RFC, but rather an example of how we could use the information provided by stages.

## Drawbacks

Making updates to past RFCs can generate a lot of  GitHub notifications for people who watch the repository. We think that the benefit of knowing when an RFC is available in a stable release outweighs this drawback. Additionally, in the future, if an app displays the frontmatter data, someone could use that as their main source of information. Lastly, the Ember Times does a great job curating RFC updates, so someone could watch it instead if the RFC repo itself has too much information.

## Alternatives/State of the Art

This section will cover how other projects have handled this type of tracking, and how we have done it in the past.

## Vue

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

 As of the writing of this RFC, there are over [70 open pull requests](https://github.com/emberjs/rfcs/pulls) for Ember's RFCs. Having so many open PRs is confusing for everyone but the core team, yet simply closing many of these PRs would not reflect the reality of the wish to bring these good ideas into Ember.

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

- If we imagine that we did this process during Octane, how would it have helped, or not?
- For RFCs that have to do with technical features, should the release version indicate when it is in Ember's blueprint, or the name and version of the package itself?
- How do we resolve disagreements over stage names? Other stage examples include: "Ready to implement," "Active," "Rejected," "Obsolete"
- Should "Under consideration" be a stage, or a GitHub label on the Pull Request? What are the pros and cons?
- Should this RFC describe the process for gathering consensus to move to Release?
- Should this RFC mention the champion at all? Should they be responsible for moving it through stages? Or was their job over when it was merged?
- Would a "postponed" stage be accurate for many of our open, stale RFCs? Should there be a postponed stage?
