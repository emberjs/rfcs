- Start Date: 2019-07-08
- Relevant Team(s): Steering, Ember.js, Ember Data, Ember CLI, Learning
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Intimate API Changes That Warrant Deprecations Should Cause Major Version Releases Upon Their Removal

## Summary

When [Ember released its first LTS version](https://blog.emberjs.com/2016/02/25/announcing-embers-first-lts.html), the concept of [editions](https://emberjs.github.io/rfcs/0364-roadmap-2018.html) did not yet exist. Major versions were responsible both for [maintaining backwards compatibility](https://semver.org/) _and_ for [marking a new paradigm](https://github.com/emberjs/rfcs/blob/9c7fe3f4e947b5f79050214334a98673494c25d7/text/0000-editions.md). Today, the latter problem belongs to editions so major version releases no longer need much ceremony. Instead, they can be relegated to marking backwards-incompatible changes.

[Intimate API](https://twitter.com/wycats/status/918644693759488005) changes already [undergo an analysis that sometimes results in deprecation messages](https://blog.emberjs.com/2016/02/25/announcing-embers-first-lts.html) before removal. I propose that in the future, any Intimate API removal that would get a deprecation warning should trigger a major version release upon its removal.

## Motivation

Ember.js has followed SemVer for many years. However, SemVer is incomplete in its definition of the term "public API." A while ago, the Ember community identified a gray-area between public and private APIs, which it dubbed "Intimate APIs." These are the parts of Ember.js that are being used by the community, but don't exist (at least completely) in its documentation.

Ember's current release process allows backwards-incompatible changes to Intimate APIs to happen in a minor release. But since the very definition of an Intimate API means that people are using it, this runs the real risk of breaking a production application. Even with the current policy of a deprecation existing for at least one LTS cycle, the following tragic scenario is possible:

> Chandra has an Ember.js application. Her company doesn't follow every best practice out there, so they don't use `package-lock.json` or `yarn.lock` to pin dependencies. Instead, their build system grabs exactly what is specified in `package.json`.
>
> Chandra reads [Ember's documentation](https://deprecations.emberjs.com/) and finds this: "Until a major revision such as 2.0 lands, code firing such deprecations is still supported by the Ember community. After the next major revision lands, the supporting code may be removed."
>
> Thus, Chandra feels safe setting `"ember-source": "^3.0.0"` in her package.json, comforted by the presence of SemVer to keep her application safe. She deploys her company's application on June 28, 2018. Her app runs on v3.2.2 and all is well. In her `beforeModel` hook, she reads `transition.targetName` (found in [the official Ember documentation](http://api.emberjs.com/ember/3.10/classes/Route/methods?anchor=resetController )) and `transition.queryParams` ([quite](http://shotgundebugging.blogspot.com/2019/02/impersonation-in-emberjs-elegantly.html) well [documented](https://codeandtechno.com/posts/user-impersonation-ember-simple-auth-doorkeeper/) by [community](https://discuss.emberjs.com/t/getting-query-params-and-segment-values-from-parent-route/6628/6) blog [posts]( https://www.tilcode.com/tag/ember-queryparams-tutorial/) and [answers](https://stackoverflow.com/a/26706095) all [over](https://stackoverflow.com/a/43310476)).
> 
> Chandra doesn't need to deploy again for a full year, as the application was developed perfectly from the start. But on July 4, 2019 the production server goes down and an automated build process checks out the repository, resolves `"ember-source": "^3.0.0"` to v3.11.0, and [her production application breaks](https://github.com/emberjs/ember.js/pull/17843)!
>
> Chandra never saw the [deprecation warnings](https://deprecations.emberjs.com/v3.x#toc_transition-state), because she never upgraded to v3.6. And now she has to leave her USA Independence Day celebration early, putting down the fireworks with family to fight metaphorical fires at work.

## Detailed design

In the current process, Intimate API changes already undergo a vetting process to see if they warrant deprecation warnings. This proposal does not suggest any changes to that process. Instead, it suggests that the outcome of the answer, "yes, this needs a deprecation," changes from, "so don't remove it until the next minor release after an LTS," to "so trigger a major version change when it releases."

This _will_ mean more major versions get released than today; but since Editions now drive paradigm shifts and marketing efforts, it shouldn't greatly affect how people talk about "what's new in Ember."

This proposal also does not suggest any changes to the community's current vocabulary. Intimate APIs are still Intimate APIs, and are considered different from Public APIs. Like today, not all Intimate API changes would warrant a deprecation warning (and therefore a major version change). Intimate APIs can still be taught as a thing to be avoided. The big change here is that in today's world, those who choose not to avoid Intimate APIs are punished by minor version upgrades. In the proposed world, they instead have to deal with breaking changes in a major version but everybody else gets that major version "for free."

Finally, this proposal also does not suggest removing deprecated public APIs on a faster (calendar-based) timeline than we currently have. Many features are marked for deprecation "until 4.0.0" -- but that's a minimum guarantee, not a maximum. If an Intimate API removal causes a version 4.0.0 to be released before the community is ready for the deprecated public features to be removed, then don't remove them yet. Nothing about SemVer requires every major release to 

## How we teach this

In a way, this is already largely what we teach publicly. Between the language on the [deprecations site](https://deprecations.emberjs.com/), and the content of the [Editions page](https://emberjs.com/editions/), this is how many people who are _not_ active participants in the Ember RFCs process might already expect things to work. And they are a large bulk of the Ember Community -- I work with people who have used Ember for years and have never heard of RFCs.

The harder part might be teaching core team members and other heavily-enfranchised individuals. Upon reading the title and/or summary of this RFC, they may get gut reactions of visceral fear that this will slow down development (it shouldn't; just release more major versions) or cause lots of teaching problems (it shouldn't; teach "octane" paradigms not "v3" paradigms).

The important thing when teaching this is to stress that _everything is the same as it has been, except that you bump a different integer when releasing any time a deprecated feature is removed_. **Whether to deprecate an Intimate API removal?** Same as before. **When to release LTS versions?** Same as before. **How many months to leave deprecated features in the framework?** Same as before. (etc.)

## Drawbacks

Currently there is a feeling of a major version release being a big deal in the Ember community. The release of v2 and v3 were moments to be celebrated, and they were markers of a different way of thinking and programming your apps. While editions are supposed to replace this, people may not be used to that idea yet. Past major versions were also a lynchpin in many users' upgrade strategies, causing people to stay on v1.13 and v2.18 much longer than any minor release. This runs the risk of that mindset sticking around for long-time users, who may be afraid to upgrade to the next major version no matter how few breaking changes there are.

Another major drawback is that there are many features that are marked as deprecated "until 4.0.0" -- the very next deprecated Intimate API removal would likely _not_ remove those deprecations, so the communication around the release of v4.0.0 (and the content on deprecations.emberjs.com) would need to be very clear about the fact that not all of these actually _did_ go away.

Finally, this proposal makes predicting when a major version will be released more difficult. Inversely, it makes predicting what major version will exist on a certain future date difficult. There may be negative externalities of this. For example, if you plan to deprecate a public feature for at least one year it's tough to pick an "until" version that's not too low; as above, too low isn't a real problem, but it still feels bad.

## Alternatives

Instead of releasing a major version when a deprecated Intimate API is removed, we could avoid removing them until the next major version was originally planned to be released. The major problem with this alternative is that it really _could_ slow down development in the framework. These APIs are being removed for good reasons, and delaying their removal delays that value.
