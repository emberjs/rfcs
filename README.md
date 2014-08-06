# Ember RFCs

> Hat tip to the [Rust RFC process].

Many changes, including bug fixes and documentation improvements can be 
implemented and reviewed via the normal GitHub pull request workflow.

Some changes though are "substantial", and we ask that these be put 
through a bit of a design process and produce a consensus among the Ember
core team.

The "RFC" (request for comments) process is intended to provide a
consistent and controlled path for new features to enter the framework.

## When you need to follow this process

You need to follow this process if you intend to make "substantial" 
changes to Ember, Ember Data or its documentation. What constitutes a
"substantial" change is evolving based on community norms, but may
include the following.

   - Any new feature that creates new API surface area, and would
     require a [feature flag] if introduced.
   - Removing features that already shipped as part of the release
     channel.


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

## What the process is

In short, to get a major feature added to Ember, one must first get the 
RFC merged into the RFC repo as a markdown file. At that point the RFC 
is 'active' and may be implemented with the goal of eventual inclusion 
into Ember.

* Fork the RFC repo http://github.com/emberjs/rfcs
* Copy `0000-template.md` to `active/0000-my-feature.md` (where 
'my-feature' is descriptive. don't assign an RFC number yet).
* Fill in the RFC
* Submit a pull request. The pull request is the time to get review of 
the design from the core team and the community.
* Build consensus and integrate feedback. RFCs that have broad support 
are much more likely to make progress than those that don't receive any 
comments.
* Eventually, somebody on the [core team] will either accept the RFC by 
merging the pull request and assigning the RFC a number, at which point 
the RFC is 'active', or reject it by closing the pull request.

## The RFC life-cycle

Once an RFC becomes active then authors may implement it and submit the 
feature as a pull request to the Ember repo. An 'active' is not a rubber 
stamp, and in particular still does not mean the feature will ultimately 
be merged; it does mean that the core team has agreed to it in principle
and are amenable to merging it.

Furthermore, the fact that a given RFC has been accepted and is
'active' implies nothing about what priority is assigned to its
implementation, nor whether anybody is currently working on it.

Modifications to active RFC's can be done in followup PR's.  We strive
to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect
every merged RFC to actually reflect what the end result will be at
the time of the next major release; therefore we try to keep each RFC
document somewhat in sync with the language feature as planned,
tracking such changes via followup pull requests to the document.

An RFC that makes it through the entire process to implementation is
considered 'complete' and is moved to the 'complete' folder; an RFC
that fails after becoming active is 'inactive' and moves to the
'inactive' folder.

## Implementing an RFC

The author of an RFC is not obligated to implement it. Of course, the
RFC author (like any other developer) is welcome to post an
implementation for review after the RFC has been accepted.

If you are interested in working on the implementation for an 'active'
RFC, but cannot determine if someone else is already working on it,
feel free to ask (e.g. by leaving a comment on the associated issue).

## Reviewing RFC's

Each week the [core team] will attempt to review some set of open RFC
pull requests.

We try to make sure that any RFC that we accept is accepted at the
Friday team meeting, and reported in the weekly blog post. Every
accepted feature should have a core team champion, who will represent
the feature and its progress.

[Rust RFC process]: https://github.com/rust-lang/rfcs
[core team]: http://emberjs.com/team/
[feature flag]: http://emberjs.com/guides/contributing/adding-new-features/
