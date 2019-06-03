# Ember RFCs

Many changes, including bug fixes and documentation improvements can be
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put
through a bit of a design process and produce a consensus among the Ember
core teams.

The "RFC" (request for comments) process is intended to provide a
consistent and controlled path for new features to enter the framework.

[Active RFC List](https://github.com/emberjs/rfcs/pulls)

## When you need to follow this process

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

## Gathering feedback before submitting

It's often helpful to get feedback on your concept before diving into the
level of API design detail required for an RFC. **You may open an
issue on this repo to start a high-level discussion**, with the goal of
eventually formulating an RFC pull request with the specific implementation
 design. We also highly recommend sharing drafts of RFCs in `#dev-rfc` on 
the [Ember Discord](https://discord.gg/emberjs) for early feedback.

## The process

In short, to get a major feature added to Ember, one must first get the
RFC merged into the RFC repo as a markdown file. At that point the RFC
is 'active' and may be implemented with the goal of eventual inclusion
into Ember.

* Fork the RFC repo http://github.com/emberjs/rfcs
* Copy the appropriate template. For most RFCs, this is `0000-template.md`, 
for deprecation RFCs it is `deprecation-template.md`.
Copy the template file to `text/0000-my-feature.md`, where
'my-feature' is descriptive. Don't assign an RFC number yet.
* Fill in the RFC. Put care into the details: **RFCs that do not
present convincing motivation, demonstrate understanding of the
impact of the design, or are disingenuous about the drawbacks or
alternatives tend to be poorly-received**.
* Fill in the relevant core teams. Use the table below to map from projects to 
teams.
* Submit a pull request. As a pull request the RFC will receive design
feedback from the larger community, and the author should be prepared
to revise it in response.
* Find a champion on the relevant core team. The champion is responsible for 
shepherding the RFC through the RFC process and representing it in core team 
meetings.
* Update the pull request to add the number of the PR to the filename and 
add a link to the PR in the header of the RFC.
* Build consensus and integrate feedback. RFCs that have broad support
are much more likely to make progress than those that don't receive any
comments.
* Eventually, the [core teams] will decide whether the RFC is a candidate
for inclusion in Ember.
* RFCs that are candidates for inclusion in Ember will enter a "final comment
period" lasting 7 days. The beginning of this period will be signaled with a
comment and tag on the RFC's pull request. Furthermore,
[Ember's official Twitter account](https://twitter.com/emberjs) will post a
tweet about the RFC to attract the community's attention.
* An RFC can be modified based upon feedback from the [core teams] and community.
Significant modifications may trigger a new final comment period.
* An RFC may be rejected by the [core teams] after public discussion has settled
and comments have been made summarizing the rationale for rejection. The RFC 
will enter a "final comment period to close" lasting 7 days. At the end of the 
"FCP to close" period, the PR will be closed.
* An RFC may also be closed by the core teams if it is superseded by a merged
RFC. In this case, a link to the new RFC should be added in a comment.
* An RFC author may withdraw their own RFC by closing it themselves.
* An RFC may be accepted at the close of its final comment period. A [core team]
member will merge the RFC's associated pull request, at which point the RFC will
become 'active'.

### Relevant Teams 

The RFC template requires indicating the relevant core teams. The following table 
offers a reference of teams responsible for each project. Please reach out for 
further guidance. 

|   Core Team   |    Project/Topics                                            |
|---------------|--------------------------------------------------------------|
| Ember.js      | Ember.js                                                     |
| Ember Data    | Ember Data                                                   |
| Ember CLI     | Ember CLI                                                    |
| Learning      | Documentation, Website, learning experiences                 |
| Steering      | Governance                                                   |
 
### Finding a champion

The RFC Process requires finding a champion from the relevant core teams. The 
champion is responsible for representing the RFC in team meetings, and for 
shepherding its progress. [Read more about the Champion's job](#champion-responsibilities)
 
- Discord
The `dev-rfc` channel on the [Ember Discord](https://discord.gg/emberjs) is 
reserved for the discussion of RFCs.
We highly recommend circulating early drafts of your RFC in this channel to both 
receive early feedback and to find a champion.  

- Request on an issue in the RFC repo or on the RFC
We monitor the RFC repository. We will circulate requests for champions but highly 
recommend discussing the RFC in Discord.  

## The RFC life-cycle

Once an RFC becomes active the relevant teams will plan the feature and create 
issues in the relevant repositories.
Becoming 'active' is not a rubber stamp, and in particular still does not mean 
the feature will ultimately be merged; it does mean that the core team has agreed 
to it in principle and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is
'active' implies nothing about what priority is assigned to its
implementation, nor whether anybody is currently working on it.

Modifications to active RFC's can be done in followup PR's. We strive
to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect
every merged RFC to actually reflect what the end result will be at
the time of the next major release; therefore we try to keep each RFC
document somewhat in sync with the feature as planned,
tracking such changes via followup pull requests to the document.

## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the
RFC author (like any other developer) is welcome to post an
implementation for review after the RFC has been accepted.

If you are interested in working on the implementation for an 'active'
RFC, but cannot determine if someone else is already working on it,
feel free to ask (e.g. by leaving a comment on the associated issue).

## For Core Team Members

### Reviewing RFCs

Each core team is responsible for reviewing open RFCs. The team must ensure 
that if an RFC is relevant to their team's responsibilities the team is 
correctly specified in the 'Relevant Team(s)' section of the RFC front-matter.
The team must also ensure that each RFC addresses any consequences, changes, or
work required in the team's area of responsibility.

As it is with the wider community, the RFC process is the time for 
teams and team members to push back on, encourage, refine, or otherwise comment 
on proposals.

### Referencing RFCs

- When mentioning RFCs that have been merged, link to the merged version, 
not to the pull-request.

### Champion Responsibilities

* achieving consensus from the team(s) to move the RFC through the stages of 
the RFC process.
* ensuring the RFC follows the RFC process.
* shepherding the planning and implementation of the RFC. Before the RFC is 
accepted, the champion may remove themselves. The champion may find a replacement 
champion at any time.

### Helpful checklists for Champions

#### Becoming champion of an RFC
- [ ] Assign the RFC to yourself

#### Moving to FCP to Merge
- [ ] Achieve consensus to move to "FCP to Merge" from relevant core teams
- [ ] Comment in the RFC to address any outstanding issues and to proclaim the 
start of the FCP period
- [ ] Tweet from `@emberjs` about the FCP 
- [ ] Ensure the RFC has had the filename and header updated with the PR number 

#### Move to FCP to Close
- [ ] Achieve consensus to move to "FCP to Close" from relevant core teams
- [ ] Comment in the RFC to explain the decision

#### Closing an RFC
- [ ] Comment about the end of the FCP period with no new info
- [ ] Close the PR

#### Merging an RFC
- [ ] Achieve consensus to merge from relevant core teams
- [ ] Ensure the RFC has had the filename and header updated with the PR number 
- [ ] Create a tracking card for the RFC implementation at {projects}
- [ ] Update the RFC header with a link to the tracking
- [ ] Merge
- [ ] Update the RFC PR with a link to the merged RFC (The `Rendered` links often
go stale when the branch or fork is deleted)
- [ ] Ensure relevant teams plan out what is necessary to implement
- [ ] Put relevant issues on the tracking

**Ember's RFC process owes its inspiration to the [Rust RFC process]**

[Rust RFC process]: https://github.com/rust-lang/rfcs
[core team]: http://emberjs.com/team/
[feature flag]: http://emberjs.com/guides/contributing/adding-new-features/
[core team notes]: https://github.com/emberjs/core-notes/tree/master/ember.js
