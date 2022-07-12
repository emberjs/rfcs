---
Stage: Accepted
Start Date: 2022-07-12
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): "Steering, Ember.js, Learning"
RFC PR: https://github.com/emberjs/rfcs/pull/830
---

# Evolving Ember's Major Version Process

## Summary <!-- omit in toc -->

Introduce a standard release train for *major* releases, analogous to Ember's release train for minor releases:

- After every `X.8` minor release, Ember will ship a new major, `(X+1).0`, which removes any deprecated code targeted for that major release.
- Deprecations targeting the next major cannot be introduced later than the `X.7` release.
- There will be a planned 12 week cycle for the transition between an `X.8` and an `(X+1).0` release to allow for stabilizing the `X.*` LTS release while working on the `(X+1).0` release.

**The result is a steady 60-week cadence between Ember major releases, complementing our steady 6-week cadence between Ember minor releases.**

How this would work in practice for Ember's lifecycle from 4.8 to 7.0

| Ember Release |    Approximate date    |
| ------------- | ---------------------- |
| 4.8           | mid October 2022       |
| 4.8 LTS       | late November 2022     |
| **5.0**       | **early January 2023** |
| 5.4           | late June 2023         |
| 5.4 LTS       | early August 2023      |
| 5.8           | early December 2023    |
| 5.8 LTS       | late January 2024      |
| **6.0**       | **early March 2024**   |
| 6.4           | mid August 2024        |
| 6.4 LTS       | late September 2024    |
| 6.8           | early February 2025    |
| 6.8 LTS       | mid March 2025         |
| **7.0**       | **late April 2025**    |


### Note <!-- omit in toc -->

There are two related considerations in this space:

- How does this relate to the lockstep versioning policy Ember, Ember CLI, and Ember Data have historically followed?
- What do we expect the timing of Ember Polaris to be relative to this cadence? More generally, exactly how should Editions relate to major releases?

However, those questions are orthogonal to this proposal: we can maintain lockstep, or not, with a standard cadence for major releases; and we can also decide whether (and if so, how) to align editions with major releases whatever cadence majors come on.


### Outline <!-- omit in toc -->

- [Motivation](#motivation)
- [Detailed design](#detailed-design)
  - [Freeze deprecations at `X.7`](#freeze-deprecations-at-x7)
  - [Formalize a 12-week gap between `X.8` and `(X+1).0`](#formalize-a-12-week-gap-between-x8-and-x10)
  - [Bootstrap with Ember v5](#bootstrap-with-ember-v5)
    - [Ember v5 deprecations](#ember-v5-deprecations)
    - [Ember CLI v5 deprecations](#ember-cli-v5-deprecations)
    - [Ember Data v5 deprecations](#ember-data-v5-deprecations)
  - [Prior art](#prior-art)
- [How we teach this](#how-we-teach-this)
  - [Publicize the change](#publicize-the-change)
  - [Website 'Releases' page](#website-releases-page)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [No change](#no-change)
  - [Pick a different cadence](#pick-a-different-cadence)
  - [Different point for freezing deprecations](#different-point-for-freezing-deprecations)
- [Unresolved Questions](#unresolved-questions)


## Motivation

Ember's approach to release versioning has served us well for the last seven years. We target minor releases every six weeks, and then do a major release to remove deprecated code on some long interval. Major releases being rare has been an explicit goal. Per [the Releases page][releases-current]:

>   You might notice that although Ember has been around for a long time, it's version number is low.
> That is because Ember aims to ship new features in minor releases, and make
  major releases as rare as possible.
> When you see a library or framework that has <i>many</i> major versions, each one of those numbers
  represents a "breaking change."
> Breaking changes force development teams to spend time researching
  the changes and modifing their codebase before they can upgrade.
> The bigger the codebase, or the more complex the app, the more time and effort it takes.
> Ember is committed to providing a better experience than that.

[releases-current]: https://github.com/ember-learn/ember-website/blob/a7a4191dc76170c855c9a0e93108c75456cdc554/app/templates/releases/index.hbs#L74-L84

The result is a steadily increasing time between major releases:[^pandemic]

| Transition |   Time    |  Start of major   | Start of next major |
| ---------- | --------- | ----------------- | ------------------- |
| 1.0 → 2.0  | 2 years   | August 13, 2013   | August 13, 2015     |
| 2.0 → 3.0  | ~2½ years | August 13, 2015   | February 14, 2018   |
| 3.0 → 4.0  | ~4 years  | February 14, 2018 | December 20, 2021   |

This is as we would expect given the explicit goal to "make major releases as rare as possible", especially given how the framework has matured such that it has not *needed* breaking changes as often. However, this policy of minimizing the number of major releases has actually tended over time to make major upgrades *more* rather than less painful—for both Ember users and Ember maintainers.[^maintainers]

- **For users:** Since each release major release comes after a longer period of time, it includes more deprecations to remove. This means that while the major releases may be more rare, they actually become *harder* over time. They also are not predictable, and therefore are difficult to account for in planning—unlike minor and LTS releases, which are *very* predictable!

- **For maintainers:** There is planning and coordination required for a major release, as well as a great deal of removal of deprecated code. This currently happens on an _ad hoc_ basis when the Framework team decides it is time. However, that means that the amount of planning work can grow very large, and the amount of work involved in removing deprecated code can be more or less unbounded. This is hard!

Switching to a predictable cadence will address both of these problems. In particular, it means that a deprecation targeting a given release will come with an expected *timeline*. Seeing that a deprecation targets v6 will tell someone that it will be removed in roughly March 2024, with the first LTS release in August 2024. That is incredibly valuable for medium- and long-term planning.

Additionally, removing deprecated code regularly decreases the surface area of the framework. This has a number of benefits:

- The API surface area is smaller, making it easier for users to know what they should use and what they should not.

- There is less code shipped in a bundle (though we also plan to address this by making Ember fully tree-shake-able and implementing "deprecation shaking").

- Introducing new features and fixing bugs are both easier, because there are fewer legacy interactions to account for over time.

- Related, new contributors can onboard more easily, because there are fewer crufty old parts of the code base to understand.

Finally, doing releases on a predictable cadence/release train also has the same effect on *major* releases that it does on *minor* releases: it decreases the pressure to get deprecations in before a major release, because there will always be another major release—and it won't be that long! It also means that the "freeze" period doesn't become a worry: it only freezes deprecations targeting the *next* major. This also enables longer-targeted deprecations where it makes sense.

For example: imagine a deprecation of some major piece of Ember landing early in 5.x (maybe deprecating the classic router in favor of some hypothetical new router design). Knowing the release cadence means that can *intentionally* target removal in 7.0 instead of 6.0. This will allow the deprecation to do its work as a messaging signal ("Don't use this in new code, and start migrating to the new thing"), while also providing a predictable and *long-term* schedule for its removal.


[^pandemic]: Even granting that the pandemic likely increased the time between 3.0 and 4.0 somewhat, we would likely have had *at least* a 2½–3-year gap between those releases.

[^maintainers]: We always prioritize users over maintainers if there is a hard choice between the two, but we also care about the long-term sustainability of the project. Things that make maintainers' lives hard decrease the sustainability of the project over time!


## Detailed design


### Freeze deprecations at `X.7`

One of the major things we learned from the 1.13 → 2.0 transition was that having too many deprecations piled up at the end of a major release can *feel* like it violates Ember's commitment to "stability without stagnation". Since then, we have frozen deprecations targeting the next major release late in the previous major release cycle. However, there have been two ongoing issues with this:

- Users have not known *when* that freeze would come. This made it difficult to know when to target a deprecation, or whether a deprecation was valuable at a given point in time, since it wasn't clear when the next major would even be.

- There has been a tendency to rush to get in deprecations in the period just before the freeze (in large part because major releases have been unpredictable: if you don't get it in *now*, it may be *years* before the deprecation takes effect!), which again undermines.

Given a known major release cadence, we can explicitly target a specific release as the last allowed time to target a given major. However, we can *also* keep landing deprecations past that freeze: they just have to target the next major. You could, for example, land a deprecation in 4.8 targeting 6.0, or even 7.0!


### Formalize a 12-week gap between `X.8` and `(X+1).0`

Major releases have sometimes taken longer than the targeted 6-week cadence to release. For example, there was an 8 week gap between 1.13 and 2.0, a 6 week gap between 2.18 and 3.0, and a 14-week gap between the releases of Ember 3.28 and Ember 4.0. We should embrace the that a major release is likely to be more work than a normal minor release, and make it *easy* to hit the cadence.

Accordingly, introduce an extra 6-week buffer for the major release. This also has the happy effect of making the release cadence super easy to explain and remember: 6 weeks between minor releases, 60 weeks between major releases.


### Bootstrap with Ember v5

When Ember adopted its 6-week "release train" cadence for minor releases, it used the 1.1 release to "bootstrap" the process and help the team learn how to do it, identify gaps, etc., with no features added. We should do the same here with 5.0. There are *very* few deprecations currently targeting Ember v5, so we can use this in a similar way.


#### Ember v5 deprecations

- [`Ember.assign`](https://deprecations.emberjs.com/v4.x#toc_ember-polyfills-deprecate-assign), replaced by `Object.assign` or object spread
- [implicit injections](https://deprecations.emberjs.com/v4.x#toc_implicit-injections) (already a no-op and do nothing, and just need to be removed)
- [AutoLocation class](https://deprecations.emberjs.com/v4.x#toc_deprecate-auto-location), switching to explicitly using `'history'` or `'hash'` as appropriate


#### Ember CLI v5 deprecations

- removal of Bower (via a number of separate feature deprecations)
- deprecate "blacklist" and "whitelist" build options


#### Ember Data v5 deprecations

N/A: none exist at the time of authoring.


### Prior art

- Angular uses a six-month cadence for major releases, with every other major release (the even-numbered releases) being an LTS release.

- Node uses a six-month cadence for major releases, with every other major release (the even-numbered releases) being an LTS release.

- TypeScript uses a quarterly release cadence, and releases a "major" once they  hit the `X.9` release. Notably, they do not use SemVer, so this is not a particularly relevant comparison for *how* we approach this. It is relevant mostly by way of providing another example of a standard cadence for releases.


## How we teach this

### Publicize the change

When this RFC is accepted, we will publish a blog post to the Ember.js blog, tweet about it from the official account, and post the update to the official Discord and Discourse instances. We may also consider publishing in other media about it (e.g. publishing a dedicated video to the Ember Videos channel on YouTube).

Additionally, when we release the final minor version of the 4.x series and again when we release 5.0, we should explicitly describe the updated policy. This is similar to how Ember approached the introduction of the "release train" with 1.x and the messaging around major releases at the 2.0 release.


### Website 'Releases' page

Update the **Our Goals** section:

- Update this bullet point to include major releases:

    > - Make a minor release about every six weeks, so teams that use Ember can plan their work

    Updated:

    > - Make a minor release about every six weeks and a major release about every sixty weeks, so teams that use Ember can plan their work

- Wholly reframe this bullet point:

    > - Only cut a new major version (i.e. make a breaking change) when we really, really have to

    Instead of focusing on duration, emphasize that we don't make breaking changes lightly:

    > - Only make breaking changes when we really, really have to

Update the **How Ember uses SemVer** section to clarify that we want to make breaking changes both rare *and predictable* and therefore manageable. We should also take this as an opportunity to clarify how we use major releases and editions.

The previous text:

> You might notice that although Ember has been around for a long time, it's version number is low. That is because Ember aims to ship new features in minor releases, and make major releases as rare as possible. When you see a library or framework that has many major versions, each one of those numbers represents a "breaking change." Breaking changes force development teams to spend time researching the changes and modifing their codebase before they can upgrade. The bigger the codebase, or the more complex the app, the more time and effort it takes. Ember is committed to providing a better experience than that.

Updated:

> Ember aims to ship new features in minor releases, to make breaking changes rare, and to make major releases predictable. Breaking changes force development teams to spend time researching the changes and modifing their codebase before they can upgrade. The bigger the codebase, or the more complex the app, the more time and effort it takes. Ember is committed to providing a better experience than that:
>
> 1. **We never couple the addition of new features to breaking changes.** Instead, we introduce a new feature to replace an existing feature, provide a migration path, then sometime later deprecate the old feature, and finally remove the old feature in a later major release.
>
> 2. **Ember major versions only remove deprecated features. They never introduce new features.** This means major releases are not exciting, just a predictable point where some cleanup happens.
>
> 3. **Ember's big releases are "Editions.** An Edition lands in a minor release and is therefore always backwards compatible. It represents the point where all the features we shipped in minor releases are polished, well-documented, and recommended for everyone to use. [Read more here.](https://emberjs.com/editions/)


## Drawbacks

- We must use this as a way of *getting better* at doing major releases. As we have done them to date, this would be incredibly stressful.

- If we land too many large deprecations in a major release, it will still cause painful churn for both users and maintainers.

- If we do not communicate clearly about this, it could surprise or confuse both existing and potential new Ember users.


## Alternatives


### No change

Keep our existing approach, releasing new majors rarely. Focus *all* of our efforts on "deprecation shaking." That solves two parts of the motivation: allowing users not to pay for deprecated code in their bundle which they aren't using, and allowing Ember to signal clearly that users *should* stop using a given pattern. The rest of the motivation would be unaddressed, though.


### Pick a different cadence

We could adopt a time-based cadence: the proposed 60-week cadence takes just *over* a year, so the cycle drifts through following years, and does not align *especially* well with most teams' planning processes. Aligning major releases to something which predictably fits year or quarter boundaries could be helpful.

- **Annual:**  This would involve doing a major release just 4 weeks after the `X.8` release, which is a very compressed timeline, so we might need to instead target `X.7` as an LTS, followed by a 10-week cycle for the major release. Alternatively, we could shift to 8-week release cycles and update our LTS cadence accordingly.

- **18- or 24-month cycles:** the same basic idea as an annual cycle, just longer. Similar issues apply for aligning the timing (though see below).

Both of those have the downside of requiring *additional* changes to our existing process and communications.

We could also keep the idea of this RFC, but pick a longer cadence:

- **11 minor releases:** This would see us do e.g. 4.8 LTS, 4.11 LTS, 5.0, etc. This would make the release cycle just a bit longer, and slot in at *exactly* an 18-month cadence if we kept the extra long (12-week) cycle for a major.

- **12 minor releases:** This is the other way of getting to an 18-month cadence for major releases, *if* we maintain the requirement that there be only 6 weeks between releases. (Keeping it to only six weeks may be easier than it was for e.g. 3.28 → 4.0, given a predictable *and* a shorter release cadence.)

- **an even longer cycle:** 16 minor releases, 24 minor releases, etc. That would still make the releases *predictable*, but we would still pile up deprecations for longer, and the work for both users and maintainers at the release cycle would still be higher. Additionally, we know from experience in software in general (especially around "devops") that the longer the cycles involved in any given process, the harder they tend to be. A higher cadence for releases actually tends to be *easier*.

That might suggest picking an even shorter cadence: new majors after every LTS release, e.g. 5.0, 5.4 LTS, 6.0, etc. That *would* help with the planning and we would get quite good at the process. There is also precedent, in that this is how both Angular and Node work. However, Angular and Node also *constantly* have LTS releases in active support across major releases, which ups the maintenance burden significantly. Picking a timeline of approximately a year (2 LTS releases) seems to best balance these tradeoffs.


### Different point for freezing deprecations

The `X.7` release is chosen as a point late in the cycle, and matches the point we have frozen deprecations relative to the next major in the pas. It is also relatively arbitrary, though! By taking the pressure off of any given major release, it's also less important. Our only *hard* constraint is that new deprecations targeting the next major not be introduced in the final LTS, so we could equally well choose `X.6`. And of course, if we picked a different cadence, we would pick a different value here as well.


## Unresolved Questions

- Given the intent to use Ember v5 for a "bootstrapping" process, should we freeze targeting deprecations for it early (say, as soon as this RFC is merged), to avoid making it more painful?
