---
Start Date: 2018-02-04
Relevant Team(s): Steering
RFC PR: https://github.com/emberjs/rfcs/pull/300
Tracking: https://github.com/emberjs/rfc-tracking/issues/1

---

# RFC (Request for Comments) Process Update

## Summary

Refine the Ember RFC process and have it apply to all Ember teams. 

## Motivation

The Ember community has been using the RFC process to great effect over the last few years.
Proposals by both Core and community members are discussed and refined
with the result coming out much stronger.

During this time, the community and the core teams have identified shortcomings 
of the RFC process as well as new requirements, which this RFC intends to address:

### Confusion between emberjs/rfcs and ember-cli/rfcs

The Ember project currently has two separate RFC processes for Ember.js and Ember CLI.

This leads to confusion because the community needs to keep track of two different repositories.
For contributors there is the overhead of having to decide where to file their RFC if the proposal involves both projects,
as well as being aware of the differences in the processes.

### The process does not cover the entire project

RFCs to emberjs/rfcs and ember-cli/rfcs have traditionally concerned themselves with features or deprecations to Ember.js and Ember CLI respectfully, with some Ember Data proposals in emberjs/rfcs.

We have already begun to use emberjs/rfcs for other initiatives, such as the project-wide Ember.js 2018 Roadmap but have not codified or updated the process to make it clear that it should be used for efforts such as a website redesign,
information architecture suggestions, SEO suggestions, and the like.

### Lingering RFCs

Both the emberjs/rfcs and the ember-cli/rfcs repositories have many open issues and pull-requests.
A percentage of these have not been active in the recent past.

We have kept PRs and issues open so people could more easily find the discussions,
but this has instead given a negative impression of staleness, as RFCs linger open without new feedback.

### The process for an RFC after it has been accepted

At the moment the process does not specify what happens when an RFC is accepted and merged. This has led to many questions about the status of merged RFCs.
  
## Detailed design

### One RFC Process for all of Ember

Ember is [organized into teams](https://emberjs.com/team/), with each team being responsible for certain projects. 
The RFC process will be a useful tool for all of those projects. 
The header of the RFC template will be updated to include a spot to specify the relevant team(s). The header will have "Ember Issue:" removed.

A list of the teams and respective projects will be added to the instructions, 
possibly with the addition of per-team instructions on specifics of the project.
Additional templates might be created as well, such a design work template.

Each team will be responsible for reviewing new RFCs and, if an RFC requires work from their team, ensuring that the RFC reflects that. 
As it is with the wider community, the RFC process is the time for teams and team members to push back on, encourage, refine, or otherwise comment on proposals.

### Require a Core Champion

To make sure that RFCs receive adequate support from the team, Ember CLI has implemented the idea of a champion associated with each RFC.
One goal is that in seeking a champion from the team,
the RFC author starts a dialogue with the team and gets some early feedback.
That champion is then responsible for representing the RFC in team meetings, and for shepherding its progress. We will import a version of this process to emberjs/rfcs:

Each RFC will require a champion from the primary core team to which the RFC has been marked relevant. 
The champion must be found by the opener of the RFC or other community member. They are not assigned by the core teams.
The champion will assign themselves on the RFC on Github. 
The champion will be responsible for:
 - achieving consensus from the team(s) to move the RFC through the stages of the RFC process.
 - ensuring the RFC follows the RFC process. 
 - shepherding the planning and implementation of the RFC.
Before the RFC is accepted, the champion may remove themselves.
The champion may find a replacement champion at any time. 

A section on 'Finding a champion' will be added to the instructions on proposing an RFC. 

### Introduce the concept of "FCP to close"

To address the problem of RFC triage and inactivity, this RFC introduces the concept of FCP to close.

Closing an RFC should be viewed as another triage tool, not as a rejection of the RFC.
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

We will have a single repository for all Ember Project RFCs.

To achieve merging ember-cli/rfcs into emberjs/rfcs the following will be done:

- Add to the RFC header to indicate it applies to ember-cli
- Copy both active and completed RFC files into `text` of emberjs/rfcs
- Transfer active PRs and Issues to emberjs/rfcs
- Archive ember-cli/rfcs

There are some concerns about links breaking when we move the files to emberjs/rfcs,
but given the fact that ember-cli/rfcs had the concept of active/completed by moving the files into different folders,
links were already being broken.

The ember-cli/rfcs do not need name or numbering changes, as there is currently no duplicated name.
Going forward, the numbering should be unified by virtue of having a single repository.

### Track RFCs after they are accepted

At the moment it is not clear what happens to an RFC after it has been merged. 

This RFC proposes that after an RFC is merged, the relevant teams, guided by the champion, 
will plan implementation by creating tracking issues in the relevant projects.

This RFC proposes having a single place to track the implementation of each RFC. 
Each RFC will have a header `Tracking:` that will be filled out with a link. At that link all issues related to that RFC, across all projects and organizations, will be enumerated.    

## How We Teach This

To ensure that contributors are updated on the RFC process and the process is clear,
the documentation should be improved in a couple of ways.

The README will be updated to reflect process changes described in this RFC. 
We will add checklists to the instructions for each stage of the RFC process to make it very clear what needs to happen.

## Drawbacks

### Adjustment period

There are active RFCs in ember-cli/rfcs. Moving these discussions would be onerous, so they should be kept there until completion, and no new RFCs accepted.

### Permalinks to ember-cli/rfcs proposals

Moving the RFC files from ember-cli/rfcs (active or completed) to emberjs/rfcs can be seen as a breaking change, and could lead to someone linking to ember-cli/rfcs and then the RFC being updated in emberjs/rfcs. However, ember-cli/rfcs already suffers from a linking problem due to the active/completed folders, as RFCs need to be moved from one to the other even after being accepted.
This could be mitigated by introducing a warning in the RFC text directing people to the new source.

## Alternatives

None at the moment.

## Unresolved questions

None at the moment.

---

## Glossary

- **RFC**: Request For Comments. The process by which a proposal is discussed by the community and then approved by an Ember team.
- **FCP**: Final Comment Period. Period of one week at the end of which an RFC is to be accepted or rejected by an Ember team. Extended in periods of one week if new concerns are raised.
