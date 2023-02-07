---
stage: accepted
start-date: 2022-02-24T00:00:00.000Z
release-date:
release-versions:
teams:
  - framework
  - data
  - cli
  - learning
prs:
  accepted: https://github.com/emberjs/rfcs/pull/800
project-link:
---


# TypeScript Adoption Plan

## Summary <!-- omit in toc -->

[RFC #0724][rfc-0724] commits Ember to officially supporting TypeScript and articulates an overall philosophy for what official support means. This RFC defines a detailed implementation plan for officially supporting TypeScript in Ember, including:

- the SemVer policies Ember packages should adopt, following [RFC #0730][rfc-0730]:
  - supported TypeScript versions
  - the "breaking change" policy
- an Edition support policy
- how we will migrate users from depending on the `@types` definitions on DefinitelyTyped to Ember packages
- test infrastructure to catch regressions early
- updates to Ember CLI to support TypeScript
- release "channel" testing analogous to Ember's existing feature flag system for runtime code
- a basic communication plan for the rollout
- updates to our guides, API docs, and even the version release blog post announcement

…and more!

[rfc-0724]: https://github.com/emberjs/rfcs/pull/724
[rfc-0730]: https://github.com/emberjs/rfcs/pull/730


### Outline <!-- omit in toc -->

- [Motivation](#motivation)
- [Detailed design](#detailed-design)
  - [Semantic Versioning](#semantic-versioning)
    - [Strictness](#strictness)
    - [Primary packages](#primary-packages)
    - [Secondary packages](#secondary-packages)
  - [Edition support policy](#edition-support-policy)
  - [RFC process updates](#rfc-process-updates)
  - [Migration from DefinitelyTyped](#migration-from-definitelytyped)
    - [Classic features](#classic-features)
    - [Ember internals](#ember-internals)
    - [Type Registries](#type-registries)
  - [CLI Integration](#cli-integration)
  - [Test Infrastructure](#test-infrastructure)
  - [Publishing types](#publishing-types)
    - [Implementation](#implementation)
    - [Release channels](#release-channels)
- [How we teach this](#how-we-teach-this)
  - [Publicizing](#publicizing)
  - [Documenting SemVer](#documenting-semver)
  - [Ember’s documentation](#embers-documentation)
    - [Guides](#guides)
    - [API documentation](#api-documentation)
  - [Migration docs](#migration-docs)
    - [General migration guide](#general-migration-guide)
    - [Ember Classic features](#ember-classic-features)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Documentation](#documentation)
  - [Classic features](#classic-features-1)
  - [Semantic versioning options](#semantic-versioning-options)
  - [Documentation](#documentation-1)
- [Unresolved questions](#unresolved-questions)


## Motivation

The overall motivation for this work is the same as that in [RFC #0724][rfc-0724]:

> In sum, making TypeScript an officially supported language for Ember will benefit all Ember users, JavaScript and TypeScript alike; it will solve many pain points for TypeScript users that cannot otherwise be addressed; and it will close a gap for Ember compared to other frameworks.

Where that RFC was concerned with *whether* we should pursue those goals, this RFC is concerned with *how* we accomplish them. The specific approach described here aims to roll out TypeScript support in a way that:

- maintains Ember’s strong stability guarantees
- allows us to make steady incremental progress
- communicates progress clearly to the community
- minimizes the migration costs for existing Ember TypeScript users


## Detailed design

To fully support TypeScript across the Ember ecosystem, we need:

- a Semantic Versioning policy which can absorb breaking changes in TypeScript itself *without* imposing breaking changes on Ember developers
- an edition support policy which tells us what our types *must* support (as distinct from what they *may* support)
- a plan for migrating existing Ember TypeScript users from DefinitelyTyped to using Ember’s own core types
- a plan for CLI integration (e.g. `ember new --typescript`)
- test infrastructure for Ember’s types, both to prevent regressions and to catch breaking changes from TypeScript early so they can be mitigated
- release infrastructure to allow us to handle pre-release testing, feature flags, and alpha and beta releases
- updates to our documentation to include TypeScript as a first-class citizen of the ecosystem

While we need full template-aware type checking to complete our support for TypeScript, this RFC intentionally defers that consideration to a dedicated RFC to hammer out the remaining design questions around [Glint][glint].


### Semantic Versioning

Ember packages which publish types will adhere to the [Semantic Versioning for TypeScript Types][rfc-0730] proposal. See that RFC for details on strategies for mitigating breaking changes from TypeScript, and see [**How We Teach This: Documenting SemVer**](#documenting-semver) below for discussion of how we will document this at the framework level.

The 10,000-foot summary of our treatment of SemVer for TypeScript is:

> Code which type checks at any given point in a major release of Ember packages will continue to type check throughout that Ember major release for consumers who use supported TypeScript versions.

This is **the “no new red squiggles rule”:** if you use supported TypeScript versions, you will not see new “red squiggles” for type errors in your editor for Ember minor version upgrades.

The only exceptions to the “no new red squiggles” rule are:

- *Bug fixes to the types*: where we previously allowed code we should not have and which could produce runtime errors, the introduction of those red squiggles is making your code safer.
- *Rare cases involving inferred types*: very rarely, we may make a change to the types which is safe for your code at runtime, and which allows TypeScript to detect new unused code paths or similar. These will introduce “red squiggles” in your editor, but in a way which will simply allow you to clean up code you didn’t need.

    <details><summary>example of this kind of change</summary>

    Assume a library provides a function, `idFor`, which gives back the `id` for a given item. Maybe it finds an `id` field on the object and hands it back if it exists, or generates a UUID otherwise, or similar—the details are unimportant, as we only care about the type signature.

    In v1.0 of the library, it specifies that it always returns a `string` or  a `number`:

    ```ts
    declare function idFor(obj: unknown): string | number;
    ```

    In our own code, we need IDs to always be `string`s, so we do something like this:

    ```ts
    const id = idFor({ cool: 'story' });
    const usableId = typeof id === 'number' ? id.toString() : id;
    ```

    Then let’s say that the authors of `idFor` recognize that returning `string | number` is kind of annoying, and they update their API to always return `string`:

    ```ts
    function idFor(obj: unknown): string;
    ```

    This is perfectly safe from a runtime perspective. Any code which worked before will work now. However, TypeScript is smart enough to see that the `typeof` check is dead code:

    ```ts
    const id = idFor({ cool: 'story' });
    const usableId = typeof id === 'number' ? id.toString() : id;
    //                                           ^^^^^^^^❌
    ```

    TypeScript will now report `"Property 'toString' does not exist on type 'never'."` on the `id.toString()` call, because it determines that `id` can never be a `number`, which makes that entire branch dead code ([playground][id-play]). The "cleanup" work here is that you can simply *get rid of* that code, though!

    For further examples and a discussion of why we cannot work around this (e.g. with lint rules), see [**Appendix C**](https://github.com/chriskrycho/ember-rfcs/blob/semver-for-ts/text/0730-semver-for-ts.md#appendix-c-on-variance-in-typescript) in the Semantic Versioning for TypeScript Types RFC.

    </details>

[id-play]: https://www.typescriptlang.org/play?#code/CYUwxgNghgTiAEBbA9sArhBA1AjPA3gFDwnwgAeADsjAC7wBmaAdmLQJbLPzvABiNABTIARgCsAXPBYBrZsgDuzAJRSAzrRjtmAc3gAfeMzSIRIGAG5S8QgF9ChUJFgIU6TPCwAmAsVIVqOkYWNk5uXgEYYXEpWXklVXgNLV0LOwcmVg4ueAA3HEFlX2swLg0eYHgAXk8cADoIoXx4UuQIKQByDRoATw74W2U0krL6NDUoEUwASUqa2h7KEGQGCuqqmo7jU3N+gH4KutpkAGVNbR1C+CleNPtCTNCc3K8rohHmct5qzy8G-iaLWQbU63RgfQGQz8JFKnzGEymIFmPwWSxWaw2m22Zhg+0OxzOKUuRRuwDuhCAA


For details on both of these, see [Semantic Versioning for TypeScript Types RFC][rfc-0730], which explains these in detail and provides justification for especially the second case.

[rfc-0730]: https://github.com/emberjs/rfcs/pull/730


#### Strictness

In line with the [Semantic Versioning for TypeScript Types RFC][rfc-0730], both Ember projects and the blueprints generated by Ember using the new `--typescript` flag will set the following compiler options:

- `strict: true`
- `noUncheckedIndexedAccess: true`
- `esModuleInterop: false`
- `allowSyntheticDefaultImports: false`

(See discussion below under [**CLI Integration**](#cli-integration) for further details on the new `--typescript` flag`.)

Beyond this, we *may* define a set of *additional* “lint”-type strictness flags over time taking advantage of TypeScript and/or ESLint, an `ember-ultra-strict` mode, and provide tooling to expose that.

A consequence of adopting `strict: true` as a behavior is that we will be progressively catching *more* type safety issues over time. Since these changes are not gated on TypeScript major versions (that is, they may appear in *any* TypeScript release), they make cause “red squiggles” during any TypeScript upgrade. However, we still believe this is an appropriate default, for the following reasons:

1. Because of the guidelines under which the TypeScript team admits new strictness settings, errors caught by a new strictness flag will *always* represent real type safety features. Features do *not* rise to this bar instead go under a set of “lint”-style rules, which today include checks like `noUncheckedIndexedAccess`, which we *will* enable, and also `noPropertyAccessFromIndexSignature`, which we will by default *not*.

2. Because of the support policy described throughout, Ember’s *own* types will always have to check with these settings enabled, including its test suite, including its *type* test suite (as described under [**Test Infrastructure**](#test-infrastructure)). This means that users will *always* be able to update to a version of TypeScript supported by the Ember version they are using and be guaranteed that one of the following is true:

    - there *are* no new errors related to their use of Ember code

    **OR**

    - any errors which *do* appear represent real possible bugs in your *use* of Ember code

This maximizes the value offered by TypeScript for consumers of Ember’s types (as well as maximizing the benefit to Ember itself internally). Developers who want to  take advantage of the strictest type checking options available to prevent runtime errors will always be able to do so. Accordingly, we do *not* guarantee that code will type check in looser modes, especially when `strictNullChecks` is disabled. *Most* code will still work correctly, but advanced types features used in Ember sometimes rely on the ability to distinguish between `null` or `undefined` and other types, and we do not commit to avoiding those—not least because there is otherwise an exponential explosion of possible strictness combinations we would have to test.

<details><summary>On including <code>noUncheckedIndexedAccess</code></summary>

We include the `noUncheckedIndexedAccess` setting in our recommended settings because it catches one of the most significant holes in the TypeScript type system prior to its introduction: not accounting for the fact that "index"-style access to objects and arrays *always* allows arbitrary access to any index (numeric or string), whether or not it is defined. For example, consider this type (not recommended for *other* reasons, but legal):

```ts
type Anything {
  [key: string]: string;
}

function describe(anything: Anything) {
  let email = person['email'];
}
```

Here, `email` would be typed as `string`, and so TypeScript would not complain if you did something like `email.length`. But you can *call* this with an object which does *not* have an `email` field on it:

```ts
describe({ noEmailsHere: 'bwahahaha' });
```

This will cause a runtime error instead of a type error. However, TypeScript is (as of recent releases) smart enough to understand that if you *check* for `"email"` it can be *known* to be present:

```ts
function describe(anything: Anything) {
  if (anything['email']) {
    let email = anything['email']
    email.length
  }
}
```

Accordingly, we include this as part of Ember’s own strictness settings and defaults generated for users.[^should-be-strict]

</details>

[^should-be-strict]: The Typed Ember team thinks TypeScript *should* have included this under `strict: true`!


#### Primary packages

The primary packages in the ecosystem, `ember-source`, `ember-data`, and `ember-cli`, will adopt the [rolling window](https://github.com/chriskrycho/ember-rfcs/blob/semver-for-ts/text/0730-semver-for-ts.md#rolling-support-windows) policy:

> In *rolling window*, a package may declare a range of supported versions which moves over time, similar to supporting evergreen browsers.
>
> Packages using the “rolling window” policy should normally support all TypeScript versions released during the current ‘LTS’ of other core packages/runtimes/etc. they support, and drop support for them only when they drop support for that ‘LTS’, to minimize the number of major versions in the ecosystem.

This aligns the support policy for these packages with the existing policy for Node and browser versions, both of which already operate on exactly this model.

Additionally, primary packages must ensure that the rolling windows *always* overlap across LTS versions, so that upgrading a primary Ember packages does not require a *simultaneous* TypeScript upgrade. That is: Ember users should always be able to upgrade to the latest supported TypeScript version for one Ember LTS, then upgrade the Ember LTS *without* changing their TypeScript version—a “laddering up” strategy.

This “laddering up” strategy means that Ember major releases do not need a special release from the constraints about what TypeScript versions they support. Instead, the requirement that users be able to upgrade across LTS releases without also requiring a simultaneous TypeScript upgrade holds: a user should be able to upgrade from the final Ember 4.x LTS to Ember 5.4 LTS by first upgrading to the latest supported TS version on the 4.x LTS and then upgrading to Ember 5.4 LTS *without* changing their TS version.

Given the release cadences, this should not be burdensome: at most there is a nine month delay between the release of a TypeScript feature and its being able to be used in Ember.

At each minor release, Ember primary packages should add any newly-released TypeScript versions to their support matrix (just as we aim to do for Node and browsers). If there is a critical bug in TS (affecting correctness or performance) which is not fixed in the time that TS version is current, Ember can choose *not* to support it.[^precedent] This is a last resort, for bugs which TS itself later resolves. If a TS minor version introduces a breaking change, Ember will update the types to absorb the breakage.[^mechanisms] In some cases, this *may* include bug fix releases to the types for LTS versions of Ember, even if they would not be in the supported range otherwise. (This is not required, but as with runtime bugs, fixes are normally backported where possible.) Bug fixes to TypeScript types for supported versions will be treated exactly as they are for runtime code:

- Stable releases will receive bug fixes for types for the six weeks they are the current stable release.
- LTS releases will receive bug fixes for types for the roughly 6 months (4 releases) they are the current active LTS.

LTS releases do *not* automatically add TypeScript versions released during their support window to their supported versions list (just as they do not for Node versions).

Ember primary packages should display the currently supported TypeScript versions prominently in their READMEs as well as in their documentation.

Finally, as long as the Ember primary packages maintain a lockstep release cadence, they must support the *same* matrix of supported TypeScript versions.[^lockstep]

[^precedent]: This has happened in the past, though rarely: once late in the TS 2.x series and once early in the 3.x series, there were significant regressions around performance and correctness which meant a given minor release could not be supported by Ember’s types on DefinitelyTyped.

[^mechanisms]: There are a variety of mechanisms by which this can be supported, ranging from simple changes to the types in most cases to occasional need for the TypeScript `typesVersions` tooling.

[^lockstep]: If Ember primary packages ever stop using a lockstep release cadence, they must continue to use the same rolling support strategy, but would need to additionally consider the intersection of their TypeScript version support as well as their support for the other core packages. However, there is no current proposal to make such a change, and it would be the responsibility of the RFC making such a proposal to account for this.


##### Example

If Ember were to begin publishing types with Ember 4.4, the flow might look something like this, given the following assumptions:

- Ember starts out supporting TypeScript 4.5 and 4.6
- TypeScript 4.6 introduces a feature not in 4.5 which Ember wants to use
- TypeScript 4.8 has a significant regression, fixed in TypeScript 4.9, which means TS 4.8 is never supported
- TypeScript 4.9 introduces a feature not in 4.6 which Ember wants to use

| Ember version  |   Supported TypeScript versions   |
| -------------- | --------------------------------- |
| **4.4 (LTS)**  | **4.5, 4.6**                      |
| 4.5            | 4.5, 4.6                          |
| 4.6            | 4.5, 4.6, 4.7                     |
| 4.7            | 4.5, 4.6, 4.7                     |
| **4.8 (LTS)**  | **4.6, 4.7**                      |
| 4.9            | 4.6, 4.7                          |
| 4.10           | 4.6, 4.7, 4.9                     |
| 4.11           | 4.6, 4.7, 4.9                     |
| **4.12 (LTS)** | **4.6, 4.7, 4.9, 5.0**[^counting] |
| 4.13           | 4.6, 4.7, 4.9, 5.0                |
| 4.14           | 4.6, 4.7, 4.9, 5.0, 5.1           |
| 4.15           | 4.6, 4.7, 4.9, 5.0, 5.1           |
| **4.16 (LTS)** | **4.9, 5.0, 5.1, 5.2**            |

The key points to notice in this upgrade cycle:

- Regular minor versions may add support for new TS versions, if one has been released.

- Ember may also choose not to support a given TS version if it cannot reasonably do so.

- LTS versions *may* drop support for old TS versions, but are not *required* to. In this example:

    - Ember 4.8 LTS *does* drop TS 4.5.
    - Ember 4.12 LTS does *not* drop TS 4.6.
    - Ember 4.16 LTS *does* drop all versions before TS 4.9.

In this example, a team upgrading from Ember 4.4 LTS to Ember 4.8 LTS can upgrade to TypeScript 4.6, then upgrade to Ember 4.8 LTS separately. They would not need to upgrade TS at all to upgrade to Ember 4.12. Then they would need to upgrade to at least TS 4.9. Note that the 4.16 LTS release *could* drop all TS versions up to 5.0. In general, however, we will not drop support for older versions unless there is a reason to do so (e.g. 5.0 itself introduced a desirable feature).

Additionally, we *may* make best-effort fixes to later (technically unsupported) TypeScript versions for LTS releases. For example, if TypeScript 4.7 introduced a breaking change which could be fixed by types-only changes to Ember 4.4 LTS, we would accept PRs to fix it and otherwise prioritize it in accordance with its severity (just as we do for runtime issues related to browsers, Node, or other ecosystem packages we depend on, e.g. `@babel` packages).

[^counting]: As part of its rejection of SemVer, TS just rolls over from an `x.9` release to the next major `y.0`, e.g. `2.9` to `3.0` and `3.9` to `4.0`.


##### Reasoning

Like browsers and Node, TypeScript regularly introduces new features which are attractive to use. Unlike browsers or Node, however, it is impossible to “polyfill” those new features for TypeScript. We want to enable primary Ember packages to take advantage of those new features without waiting for a full major release. For example, in the almost 4-year span between the releases of Ember 3.0 (February 2018) and Ember 4.0 (December 2021), TypeScript released the following key features which dramatically changed our ability to represent Ember’s APIs correctly or to use TypeScript effectively (as well as many other smaller enhancements):

- conditional types (2.8, March 2018)
- `unknown`, spread parameter types, improved tuple types with optional and spread elements, and composite projects (3.0)
- improved strictness settings (3.2, 4.0, 4.2, 4.4)
- `const` assertions and higher-order inference for functions, which was key to enabling Glint (3.4)
- assertion functions, allowing more useful typing of `assert` (3.7)
- the `declare` modifier, allowing safe declaration of injections and CPs (3.7)
- spec compatibility for class fields, optional chaining,. and nullish coalescing (3.7)
- spec compatibility for top-level `await` (3.8)
- variadic tuple types and labeled tuple elements (4.0, improved in 4.2)
- template literal types (4.1)
- spec compatibility for `#private` class fields (4.3)

Had we been publishing types and using the *simple majors*, we would not have been able to adopt *any* of those features, because doing so would have required downstream consumers to update to versions which supported them. This is basically the same problem we had with IE11 support: any feature which could not be polyfilled, we could not use at any point in the Ember 3 era.[^polyfill]

Using the “rolling window” policy allows us to adopt new features from TypeScript in the same way we do with Node and browsers, and never couples users to do TypeScript and Ember primary package upgrades simultaneously.

[^polyfill]: In fact, we could not even use some features which *could* be polyfilled in some places, because the polyfill was too expensive!


#### Secondary packages

All secondary packages (i.e. all officially-published packages which are not listed above) in the ecosystem should use the same [simple majors](https://github.com/chriskrycho/ember-rfcs/blob/semver-for-ts/text/0730-semver-for-ts.md#simple-majors) policy

> In *simple majors*, dropping support for a previously supported compiler version requires a breaking change.

As with the primary packages, this aligns secondary packages with their existing support policies, as they already treat Node version support exactly this way.

For example: `ember-cli-htmlbars` currently publishes types, but there is no documented contract for SemVer. Following this RFC and [the SemVer for TS RFC][rfc-0730], it must:

- link to the SemVer-for-TS spec
- include the currently supported TypeScript versions in its README
- specify that it uses the *simple majors policy* in its README
- add all supported TS versions to its CI jobs
- cut a new major any time it drops a TS version from support (which, for ecosystem stability, should generally be aligned with dropping a Node version when it goes out of LTS)


### Edition support policy

Per [RFC #0724: Official TypeScript Support][rfc-0724] (emphasis added):

> All libraries which are installed as part of the default blueprint must ship accurate and up-to-date type definitions **for the current edition**. These types will uphold a Semantic Versioning commitment which includes a definition of SemVer for TypeScript types as well as a specification of supported compiler versions and settings, so that TypeScript will receive the same stability commitments as the rest of Ember.

Ember’s types will always provide full support for the APIs of the *current* edition, and is *not* required to continue supporting previous editions when crossing a major version change. This does *not* supercede the Semantic Versioning commitment, however.

What this means in practice is:

1. If a new edition is introduced during a given major release (e.g. the projected Polaris Edition during 4.x), the types will maintain support for *both* editions. This is required by the SemVer support policy: dropping support for the previous edition would be a breaking change!

2. When a new major version is released after an edition has been introduced (e.g. v4 after Octane and presumably v5 after Polaris), Ember may drop some or all support for features which are not part of the updated programming model represented by the new edition.

The motivating example here is the custom class system from Ember Classic (as discussed below under [**Migration from DefinitelyTyped: Classic Features**](#classic-features)). While this continues to exist and works, it is extremely difficult to provide types for and the value of the types is very low. Ember users are recommended to use native classes instead. Accordingly, when Ember begins publishing its own types, we will only support a very limited version of the Classic features like `.extend()`, `.reopen()`, `.reopenClass()`, and legacy decorator forms which interoperate with them.

If we did *not* adopt this option, Ember would have to provide the same extensive support for the Classic features that the DefinitelyTyped version of the types have historically provided, despite the very bad cost-benefit ratio of these types for our maintenance.

In future editions, it is conceivable that we might lean on this escape hatch to drop support for legacy CLI features (after Embroider is standard) or legacy routing features (if Ember adopts a modernized routing system) etc. However, in general we do *not* expect to need it, because our designs have generally been accounting for TypeScript even before making it an officially supported language for the framework.


### RFC process updates

RFCs which introduce new APIs must specify the public types for those APIs. RFCs which deprecate existing APIs must consider the migration path for TypeScript users as well as JavaScript users.

Note that this is not a major new design constraint, because we have largely already been doing this! This simply formalizes the process Ember has been using for the past few years already: we have regularly used TypeScript signatures to represent new APIs, and TypeScript concerns have been included in the discussion of API proposals.

This does introduce two new small considerations:

- New APIs will now need to specify import paths for type-only imports. For example, today there is no public import for `Transition` or `RouteInfo`, because they are not intended for end users to construct. However, types for them *should* be publicly exported because end users need to be able to name those types them in their own code. If we were to write [RFC #0095: Router Service](https://emberjs.github.io/rfcs/0095-router-service.html) today, we would specify imports for both.

- New APIs will need to specify whether a given interface or type is intended for user-constructibility. This is part of the Semantic Versioning contract. By way of example:

    - Both `Transition` and `RouteInfo` are *non*-user-constructible. Although they are classes under the hood, it is illegal for end users to make their own copies: the only legal way to get one is *from the framework*. We can enforce this for TypeScript users with type machinery under the hood, but it is important that they be documented accordingly as well.

    - Classes like `Service` and the Glimmer `Component` are *intended* to be user-subclassed, but are *not* intended to be implemented directly.

    - Interfaces like the `ComponentManager`, `ModifierManager`, and `HelperManager` *are* intended to be user-constructible, and in fact this is the entire reason they exist.

The Detailed Design paragraph of the default RFC template will be updated to include corresponding text:

```markdown
> This is the bulk of the RFC.

> Explain the design in enough detail for somebody
familiar with the framework to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here. New types should be specified as they are intended to be used by
TypeScript consumers, including their import locations and whether they are
intended to be constructed by end users.
```

The “Transition path” paragraph of the deprecation RFC template will be updated to include the following:

```markdown
> This is the bulk of the RFC. Explain the use-cases that deprecated functionality
covers, and for each use-case, describe the transition path.
Describe it in enough detail for someone who uses the deprecated functionality
to understand, for someone to write the deprecation guide, and for someone
familiar with the implementation to implement. Make sure to include details
specific to TypeScript users, if any.
```


### Migration from DefinitelyTyped

There are many existing Ember TypeScript users, who use the `@types` packages maintained by the Typed Ember team on [the DefinitelyTyped repository][DT] since 2017. We need to support those users migrating off of DefinitelyTyped in a minimally-disruptive way. At the same time, we *do* have an opportunity to align the types with [the edition support policy described above](#edition-support-policy), as well as to fix some early infelicities in design choices.[^infelicities]

[DT]: https://github.com/DefinitelyTyped/DefinitelyTyped

Accordingly, we will:

- simplify our handling of Classic features (see [**Migration from DefinitelyTyped: Classic features**](#classic-features) below)
- update our handling of type registries (see [**Migration from DefinitelyTyped: Type registries**](#type-registries) below)
- simplify or improve other APIs if and as we see opportunities

[^infelicities]: There are a number of reasons for those infelicities: in some cases, it was simply our own ignorance when we wrote the types; in others, TypeScript did not yet support the features we needed to represent Ember; and in yet others Ember’s own design wasn’t great (we have all started designing better JS APIs in the meantime, partly informed by experience with TS!).

To reiterate, though: in general, the goal will be to *minimize* disruption to existing users. The majority of the work will be as described in [**How we teach this: Migration docs**](#migration-docs) below.

For the current DefinitelyTyped maintainers (the Typed Ember team and some other contributors), there is one additional step: we will follow the DefinitelyTyped [guide to removing a package](https://github.com/DefinitelyTyped/DefinitelyTyped#removing-a-package), so that users can get support for package versions *up to* the point where the package was removed from `@types`, while package versions after that which try to use `@types` will be notified that they should switch.


#### Classic features

Per the edition support policy, we will provide minimal support for Ember Classic features:

- **Ember's classic class system**: we will provide minimal definitions for the `.extend()`, `.reopen()`, `.reopenClass()`, and `.create()` methods, which make no attempt to use them to actually update the types of the items they modify.

    It is not possible to provide a good experience working with those types (in particular, working with mixins *at all* requires extensive unsafe casting), and they are no longer recommended for any new code. We will support them only because they are *required* for migrating some critical ecosystem code which still depends on them (especially around Ember Data).

    <details><summary>detailed discussion of classic class system updates</summary>

    All of the relevant tweaks to class types can be represented using [declaration merging][declaration-merging]. Given a class `Demo`:

    - including a `Mixin` with `.extend()` can be represented with a type alias for the mixin `type TheMixin = ...` and `interface Demo extends TheMixin {}`
    - reopen can be represented with a type alias for the mixin `type TheReopen = ...` and `interface Demo extends TheReopen {}`
    - `reopenClass` can be represented with a namespace for the reopened type, `namespace Demo { ... }`

    Accordingly, the types for the `.extend()`, `.create()`, `.reopen()`, and `.reopenClass()` methods can become much “dumber” than in the DefinitelyTyped definitions, providing a minimal layer of type safety but using these other techniques for the transition/fallback period.

    For a full working demo, see [this playground][classic-fallback-playground].

    [classic-fallback-playground]: https://www.typescriptlang.org/play?#code/PTAEFpNBnAXBDATrUBTAtgI1Y0B7TAK1QGMVYBPAB1WgihACgATUgGyVVBI+joFkAlgA9BAO1ABvRqBgJYgkt0Sp4sVAB4ASgD4AFFUR4q0AFygtASnNbGAX0Yt2nbrzoBhPCoDyRUimlZODVFNGF1MWYNdzCI5g8vVF9iMn0ZWVBYAAtBM1B3ABp02QA6MsNjPIIU2ABtAF1063yAbnT04IUlEhU1TRjUcNRIhJ8-VL1izJy8wqmKk3Nq-ybzdzaO+VCVY2Ho2OH4-MTk-zSM6dy1oouFqvHYG9Bm9ccgraUdmjF3N33BuKjJIPc4ZbJXfJPWR3JYPJ4vNoOJw8FwAMwArmIyII8BIAOaoWAaAAqBVAAGkDiNQABrVAUPCo0DE-QIRAE2DmUm0+nmcnNYm1cn1NqsFEqUAYrEKXGgDl6Nkc8yYmliPAAdzEZLpFHMcEQ4jxzRVas1oucEql2Nl0EJJLJlIBhzoOsZzNZSCVzO1vIpZIAbvA2OjUFyhfUBeHzeKuFaZRJbUSAGoe9mE5ViVUarU83VyA1iPEBoMh8xJ5pJjYovigACiWBwpzIVKOnjGNSkSJA9HAaEiaAbuGWzcoNDokF7TEY3YnoHQInEjBIuLgoAAIhg8EJRBIALygbfiEo9VTqPSBOeErJ4ZgAMSM6EPYj0nvMYnRg+a-rwgmYUimiYKjMZIAOTzjuAAKRhUCBZKepYbSyHYTzgeIUHGOYAAMRR2AhjijlwG7oFuC57pk1CoG6REkTuGwzlA6LQIac6kYw4jqIgqLwCQhGbi2dDUU+nZLm4658U61L1tgiBNrAJQScweiCaRlj-rIYghP6qDoVQeqwAWeKgAAPqA75sGwoD7mZbCIo49G9oxzFfMMS4rigWiUd81GWWpl7ZDe954OgHm7M+4J5NRzTvoOvmyCosDoog+KEkBuSgahYg6SBeFIfY+EURYnnDN5+4EW6IVeZujjsTgXE8WJxH8YVoXeZIDjUSUznPhVxWbnh05gLOjmFqAXW-PAfCuWIq49T8bglb5qIPrNqDMONfDmPpIb2BsYocBKGnoLQVDcbxjUXoMVBeCgy7TSgS1BSta1uOYmB4HgbCqGItkDfQoCsMRU2rkoVmoOqDV4HoeEkCUGkKFpOk+QARFkqDmXgZLql4bDMAAhEjbQw0d-l3g+T56AAjAATAAzHhghMnoROkYjuO7vuRNXgFy1Fc+liqReDOgEpm4lA9wW86t63QALUy3dAH2oCUbB4HiehI4APBuAJH7SM5aADgOEAA

    </details>


- **Ember’s `get` and `set` helpers:** we will not provide types to make `get` and `set` type-safe beyond simple property lookups on objects—i.e. no support for nested path lookups.

    While it would be possible to do so using advanced type system features, doing so would have a high maintenance cost and provides very little value in the Ember v4 era, where use of native property access is possible for all features which historically required `get` and `set` and their `-Properties` variants.[^get-set]

- **Classic computed property handling:** we will not provide “safe” types for the classic form of computed properties.

    <details><summary>detailed discussion of classic computed property updates</summary>

    Since they integrate with the clsasic class system and mixins, it is impossible to provide a correct experience of them in terms of the keys they accept or the `this` type within them: the types become circular. The supplied types for them will simply accept a dumb list of strings as dependent keys and a simple callback. All access to `this` will require unsafe casts.

    We will also continue to provide public types for the decorator forms, `@computed` as well as the computed property macros like `@alias`. However, decorators currently cannot affect the type of the decorated item (a TypeScript restriction), and so the public type definitions supplied will continue to simply specify them as `PropertyDecorator`s.

    Additionally, this means that we will eliminate the `UnwrapComputedPropertyGetter`, `UnwrapComputedPropertySetter`, and related types entirely. The return type will become the normal value return tyeps.

    The result will be similar to the definitions on DefinitelyTyped, but with much less “type machinery”.

    </details>

See below for a discussion of how we should approach TypeScript with Classic features in the documentation.

[declaration-merging]: https://www.typescriptlang.org/docs/handbook/declaration-merging.html


#### Ember internals

Ember's internals will regularly include APIs which are *not* intended to be public, and should be excluded from public builds. In most cases, we can solve this by making sure our type publishing tooling understands the `@internal` annotation in comments and excludes items marked with it from publishing to the public API. In conjunction with the workflow described in [**Publishing types: Implementation**](#implementation), below, this can also allow us to publish internal-only builds for collaborators who need access to those APIs. For example, Ember’s test infrastructure lives in separate packages, but has sometimes needed access to private APIs to work.

A particularly vexing version of this problem is that Ember’s own internals still make extensive use of Ember Classic features—features we are explicitly choosing not to support by publishing rich types for them. This is well-motivated: in many cases we *must* continue to use those types internally to prevent consumers who are still using those features from being broken in surprising and hard-to-debug ways.[^zebra] Accordingly, there are *more* robust types for `.extend()`, `.reopen()`, etc. in Ember’s internals than we want to support publicly. This is also tractable, though: we can provide those internal-only extensions via [declaration merging][declaration-merging] in imports which are *not* part of the published build, and which therefore do not impact the public APIs.

The main risk here is that we need to make sure to run type tests against what is *published*, rather than from within the framework itself.

[^get-set]: This has been true for *many* things since Ember 3.1, which unlocked it for getters in the Ember Classic world; and for nearly everything since Ember Octane’s release in 3.15, which unlocked it for setters for all `@tracked` properties. With changes to Ember Data in 3.28, it also works (using native proxies) even for async relationships. Finally, its handling of possibly-`undefined` intermediate properties has been superceded by a language feature: [optional-chaining](http://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Optional_chaining).

[^zebra]: The primary example here is the “zebra-striping” problem with native classes and classic classes. When you set instance properties on a native class and then extend from it using a classic class, the values set in the subclass using `.extend()` will be superceded by the class properties on the parent native class. There are a number of other related issues around native and classic class interop; see [this Ember Twiddle][twiddle] for a demo of two of them. This “makes sense”: the fields set via `.extend()` go on the prototype while class fields are per-instance, and so override the prototype values—but it is extremely confusing in practice. Until we *remove* the ability to use `.extend()`, our internals likely need to be implemented as classic classes, or they need to use annoying hacks which amount to doing the same!

[twiddle]: https://ember-twiddle.com/fdf70756b8d551b3364fd3278f66c8a9


#### Type Registries

The existing type definitions on [DefinitelyTyped][DT] make heavy use of “type registries”, which use some advanced type system features introduced early in TypeScript 2.x to map string keys to specific types. This allowed us to support string-based lookups for many Ember Classic APIs, particularly in the dependency injection (DI) system and Ember Data lookups.

For the classic DI system, we no longer require (or benefit from) string key-based lookups to resolve the type of an injected item. In Ember Classic, we could use those key-based lookups to infer the type:

- `app/services/session.ts`

    ```ts
    import Service from '@ember/service';

    export default class SessionService extends Service {
      login(username: string, password: string) {
        // ...
      }

      logout() {
        // ...
      }
    }

    declare module '@ember/service' {
      interface Registry {
        session: SessionService;
      }
    }
    ```

- `app/components/some-component.ts` (note the odd mixed declaration form!):

    ```ts
    import Component from '@ember/component';
    import { service } from '@ember/service';
    import { action } from '@ember/object';

    export default Component.extend({
      // session here has the type SessionService...
      session: service('session'),
    }) {
      @action login(username: string, password: string) {
        // ...so this call is type safe
        this.session.login(username, string);
      }
    }
    ```

With the introduction of full support for native classes and decorators in Octane, and given the current status of the decorators implementation in TypeScript, this does not provide type inference. However, these kinds of type registries can still provide two benefits for consumers:

1. When renaming a service injection, like `@service('session') sessionService;`, or using its full name, like `@service('shared@session') session;`, the registry can still check that the resolved name is one that is registered.

2. It can be used with the other things in the DI system, e.g. making `Owner.lookup('service:session')` type safe, and thereby making things like `this.owner.lookup('service:session')` in tests automatically be well-typed.

The `@service` decorator will therefore continue to accept a string key, and will validate that against a registry, even though decorators cannot change the types of the items they decorate in TypeScript today. Additionally, the registry will be integrated into `Owner` types so that it can be used more generally. Moreover, the design for an `Owner` registry should allow others to integrate in the same way.

Controller injection, by contrast with service injection, is decreasingly used across the ecosystem and not generally recommended, and we expect it to be deprecated during Ember v6. Accordingly, we will *keep* support for the service registry while dropping support for the controller registry. (Given an appropriate design for `Owner`, end users could reimplement this if they so chose.)

Future designs for services and dependency injection more generally will need to be written to account for the capabilities of TypeScript’s implementation of Stage 3 decorators as they grow and change over time.

For Ember Data, there is considerably more ongoing need for registry-style APIs. `Store.findRecord` isn’t going anywhere any time soon, for example, and it *requires* some kind of registry to make `this.store.findRecord('user', 1)` correctly return a `User` model. Many other APIs within Ember Data similarly require registries for type safety. Thus, Ember Data will need its own dedicated design for handling those, and this RFC leaves those for the Ember Data team to address in a dedicated RFC.


### CLI Integration

We will introduce a new `--typescript` (`-ts`) flag for the `ember new` and `ember addon` commands, allowing users to opt into TypeScript when generating a new project. Using this flag will:

- Set the `isTypeScriptProject` option for `.ember-cli` (introduced in [RFC #0776][rfc-0776]) to `true`, so that blueprints are generated with TypeScript by default in the project.
- Configure linting:
    - In the`.eslintrc.js` blueprint:
        - Use `@typescript-eslint/parser` instead of the Babel ESLint parser.
        - Include `@typescript-eslint` in the `plugins` key.
        - Include `plugin:@typescript-eslint/recommended` in the `extends` key.
    - Install the `@typescript-eslint` dependencies instead of,  or in addition to (as appropriate), the Babel ESLint dependencies in `package.json`.
- Configure [Glint][glint] for type checking and, for addons, emitting type declarations during build.
    - Create `tsconfig.json` files and generate their `compilerOptions.paths`.
    - Configure `ember-cli-babel` to include the TypeScript transform. (This may just be an “out of the box” setting so it always works and does not require configuration!)
    - Set up packages with scripts to do type checking and, in the case of v1 addons, emitting type declarations using `glint`.

[rfc-0776]: https://emberjs.github.io/rfcs/0776-typescript-blueprints.html

We will also update the Glimmer Component blueprint in the Ember.js repo to include [the component’s signature][rfc-0748], so that it can be used by TypeScript.

We will be sunsetting `ember-cli-typescript` as part of this process. It was a valuable and needful part of the ecosystem when we did *not* have official support, but now it duplicates other sources of configuration and tooling, making maintenance needlessly more complicated. All of its capabilities can be managed more cleanly in other projects: `ember-source` and other addons can ship blueprints which support both TypeScript and JavaScript, `ember-cli-babel` and other build tools can manage transpilation, and `glint` supports end-to-end type checking.

[glint]: https://github.com/typed-ember/glint

We will also deprecate the `ember-cli-typescript-blueprints` repository, since it will become defunct, with the blueprints moving to the host repository (`ember-source`, `ember-data`, and `ember-cli`) and being authored in TypeScript directly, with types stripped for JS consumers, so they remain in sync permanently. (See again [RFC #0776][rfc-0776] for details.)

[rfc-0748]: https://github.com/emberjs/rfcs/pull/748


### Test Infrastructure

To meet the Semantic Versioning and stability guarantees described above, we need strong testing infrastructure in place. We need to avoid introducing breaking changes in our types (the same as in our runtime code). We also need to catch breaking changes introduced by TypeScript, so that we can mitigate them.

To this end, we will extend our existing test infrastructure in the following ways:

1. We will convert our test suite to use TypeScript, so that all existing tests can be used for catching type errors.
2. We will add dedicated “type tests” using tooling like [expect-type](https://github.com/mmkal/ts/tree/master/packages/expect-type) to guarantee we are upholding *exactly* the contract we intend for our published types.
3. We will add CI jobs which type check the code against all supported TypeScript versions.
4. We will incorporate checking against the `typescript@next` nightly builds so that we can identify breaking changes early.

Point (4) will make sure that we have early warning about any inbound breakage from TypeScript itself, generally giving us on the order of 2–3 Ember minor release cycles to prepare for those changes and prevent them from breaking Ember users.


### Publishing types

When Ember official packages begin publishing types, they will need to account for TypeScript's default of using Node module resolution to look up type definitions. Specifically, a number of Ember packages which have multiple “entry points”—that is, multiple modules in a package from which it is valid to import, like `@ember/object` and `@ember/object/computed`. The only way to make this work “automatically” today is to publish types in the root of the package, just as that was the only way to make non-root modules resolve correctly in a Node package historically. A publish step which copies the types to the root is possible, and that is how `ember-cli-typescript` has made types for addons work correctly. That is *not* desirable for Ember core packages, however, because it makes it impossible to provide things like preview access to types before stabilization, types for different editions, etc. (each discussed below).

In the future, we will be able to publish types in a much more “natural” way for Ember’s package design, by taking advantage of [Node package entry points][package-exports]. However, TypeScript support for that feature is still experimental and available only in TS nightly builds, so our design does not depend on it.

[package-exports]: https://nodejs.org/api/packages.html#package-entry-points


#### Implementation

As long as we are not publishing actual packages for the `@ember/*` and `@ember-data/*` packages, we will also need to include one additional file in our output which makes the published types visible to the consumer: a “root” entry point module for the package, published either in the root as `index.d.ts` or in some other location specified by the `types` key in `package.json`. Using `ember-source` as an example, we might publish the following to `dist/index.d.ts`, with `"types": "./dist/index.d.ts"` set in `package.json`:

```ts
import './types/ember-application.d.ts';
import './types/ember-object/index.d.ts';
import './types/ember-object/computed.d.ts';
// ...etc.
```

The referenced files in turn will have `declare module` statements in them, generated by whatever tooling we use to create public API rollups (see discussion below). The `index.d.ts` acts as a type-level “side-effect” module. It makes the `@ember/*` types *visible* to any app or addon which does `import 'ember-source';`. This will make compilation work correctly *without* configuring `compilerOptions.paths` in `tsconfig.json`.

We will accordingly incorporate that `import` statement in our blueprints (with a long comment explaining why it exists) and document it.

#### Release channels

These moves also allow us to support another constraint: As with runtime code, TypeScript types need to be provided via Ember's release channel mechanism.

1. Types for a given runtime feature should be available with the version the runtime feature appears with.
2. Types for unreleased features should be available only on `master` and in canary builds.

We can solve both (1) and (2) by using the `@alpha` and `@beta` tags in conjunction with tooling which understands how to use those to strip non-public types from the generated types.[^extractor-tool] When generating a canary build, items marked with `@alpha` will be included; but for beta and canary builds, they will be excluded. The same holds for beta vs. stable builds and the `@beta` flag.

As part of the rollout, we may also provide one *additional* level of publishing: publishing to a beta location, allowing early adopters to help prove out and find bugs with the types while not losing any *runtime* guarantees. For example: an app which tracks LTS releases for stability might nonetheless want to use the official TypeScript types, even if they are not yet published on Ember 4.4 LTS.

In that case, we will not yet be publishing a root `index.d.ts` for `ember-source` etc. in a location TypeScript will resolve automatically—not in the root, and with no corresponding `"types"` field in `package.json`. Instead, we might publish it to something like `types/experimental.d.t.s`. Early adopters could then do `import 'ember-source/types/experimental';`. This will prevent anyone from *accidentally* opting into those types, while still allowing feedback from early adopters.

This design works well with future possibilities in the design space as well:

- When TypeScript begins supporting Node’s [entry points][package-exports], we will be able to leave this in place while *also* using `exports` to make it unnecessary, allowing a smooth upgrade path. Once we *require* that version of TypeScript as the minimum supported version, we will be able to deprecate the import, and then we can drop it at the next Ember major.

- When delivering a new Edition, we *could* publish edition-specific types which allow users to opt into *only* using the new Edition, and they might do `import 'ember-source/types/polaris.d.ts`. This would be akin to how we set new defaults for linting, blueprints, etc. once the `"edition"` flag is updated for an app or addon, though stronger.

[^extractor-tool]: This RFC intentionally leaves the tool unspecified, because it is an implementation detail. Presently, it looks like our best bet will be combining multiple existing tools, for example combining [rollup-plugin-dts](https://github.com/Swatinem/rollup-plugin-dts) with [API Extractor](https://api-extractor.com). That could change over time, though!


## How we teach this

There are four major components to teaching the Ember community about this new capability:

- Publicizing it, including what it does and does not entail.
- Documenting the SemVer commitments
- Incorporating support for TypeScript into Ember’s documentation.
- Documenting the migration path for existing Ember TypeScript users from DefinitelyTyped to using Ember directly.

Note that this is in addition to the process-related updates described above in [**Detailed design: RFC process updates**](#rfc-process-updates).


### Publicizing

First, we should announce official support when it lands across Ember’s various media channels (blog, Twitter, etc.): official support for TypeScript is a big deal. If possible, we should also take the opportunity to engage with high-profile podcasts to talk about our approach and the benefits we hope it brings for both our users and for the rest of the TypeScript ecosystem.

In particular, we should emphasize:

- that our official support for TypeScript is about making it a full peer to JavaScript—*not* replacing JavaScript, and never *requiring* TypeScript, but rather supporting TypeScript users as equal peers to JavaScript users, and improving the JavaScript authoring experience using TypeScript “under the hood”

- what is new about Ember’s having official support, as compared to the community support we have had for the past few years, including:
  - the integration of TypeScript into the design process
  - the benefits of publishing types directly from Ember’s source, and thereby solving the problem of keeping Ember and its types in sync over time
  - the stability guarantees that come with a clear definition of and commitment to Semantic Versioning for TypeScript

Second, we should incorporate messaging about TypeScript support in all of our discussions of the planned Polaris edition. Along with Embroider and First-Class Component Templates (if accepted), it will represent a significant shift in the experience of authoring Ember apps.


### Documenting SemVer

Besides the per-package documentation, we will also include a discussion of how Ember handles *TypeScript* in its SemVer commitments under the **How Ember uses SemVer** section on the [Releases](https://emberjs.com/releases/) page. There, we will link to the published SemVer for Types spec as well as summarizing how end users should think about it—just as we already do for runtime. This should likely be a new subsection which extends the existing discussion, e.g. **Notes for TypeScript users** just after the existing **What SemVer means for your app** section.

Additionally, when each new version of Ember is released, any updates to supported versions should be included in the blog post associated with that version in a dedicated section at the same level as the **Ember**, **Ember Data**, and **Ember CLI** sections today. For example, in an Ember 4.6 release which added support for TypeScript 4.7, the section might look like this, immediately following the Ember CLI section:

> ### TypeScript support
>
> Using Ember does not require using TypeScript, but Ember provides first-class TypeScript integration for users who wish to use TypeScript, with strong backwards-compatibility guarantees.
>
> You can always update TypeScript to a supported version independently of updating your Ember, Ember Data, and Ember CLI versions.
>
> Ember.js, Ember Data, and Ember CLI 4.6 all **added support for TypeScript 4.7**. Supported TypeScript versions now include 4.5, 4.6, and 4.7.

In an Ember 4.8 LTS release which *dropped* support for TypeScript 4.5 (as described in [the example above](#example)), we would emphasize this in the text:

> ### TypeScript support
>
> Using Ember does not require using TypeScript, but Ember provides first-class TypeScript integration for users who wish to use TypeScript, with strong backwards-compatibility guarantees.
>
> You can always update TypeScript to a supported version independently of updating your Ember, Ember Data, and Ember CLI versions.
>
> Ember.js, Ember Data, and Ember CLI 4.6 all **added support for TypeScript 4.7** and **dropped support for TypeScript version 4.5**. Supported TypeScript versions now include 4.6, and 4.7. To upgrade to Ember 4.8, you should first upgrade to at least TypeScript 4.6.


### Ember’s documentation

Ember’s documentation needs to treat TypeScript as a first-class citizen. **We take it as a fundamental consideration that types and documentation *must* stay in sync.** In its most ambitious form, this could lead us to massively revamp Ember’s docs, so that every example can toggle between JS and TS. For prior art, see how Microsoft’s API docs support all of C#, F#, C++, and Visual Basic; or how Apple’s API docs support both Objective-C and Swift. However valuable and worthwhile such an investment would be, doing it would be a massive amount of work. Instead, this RFC recommends a more measured approach—while still encouraging further consideration of that very valuable work! It leaves the guides mostly as-is, with light additions of new content; and requires the updates *only* to API documentation, where it is a hard necessity.


#### Guides

We will add a new TypeScript-focused section to Ember’s guides, built on the lessons learned (and in some cases possibly using content directly from) the docs written for ember-cli-typescript. As with the existing guides at <docs.ember-cli-typescript.com>, the goal here will be to provide an explanation of the *additional* features and mechanics users need to know if working with TypeScript. Some key topics might include:

- the setup process, including running `--typescript` at the start of a project or manually configuring it later
- how we compile apps and addons (i.e. using Babel rather than tsc), and why
- how the `paths` mappings work and why they are necessary
- working with Ember Classic features

We should also introduce some of the material from the Typed Ember docs in dedicated TypeScript sections to each of the Core Concepts guides. For example:

- in the Components guide, we could add a section describing how to write the signature for a Glimmer Component
- in the Services guide, we could add a section show how to safely write a well-typed service injection
- etc.


#### API documentation

API documentation *must* be updated to show TypeScript type signatures. This will make it possible for TypeScript users to see in the docs exactly the same thing they see in their editor, thereby avoiding any potential confusion from documentation being out of sync with the types provided. In support of this, we will need to update our tooling to support deriving the API documentation information from the types, so that there is a single source of truth.

There are also two other new concerns for documenting types which are meant to be consumed *only* as types: where/how to import and document them, and whether they are user-constructible. I will use the `RouteInfo` type from Ember’s routing system as a motivating example here.

**Import locations:** historically, type-only imports have been documented but without *having* to pin down a fixed location. For example, as of the time of writing, `RouteInfo` is documented as belonging only to the `ember` module. This has been “fine” because users expect to *receive* `RouteInfo` instances, not construct them directly, but when needing to name the type, the type must have a definite import location (presumably `@ember/routing`).

**User-constructibility:** as [described in the SemVer for TS types RFC](https://github.com/chriskrycho/ember-rfcs/blob/semver-for-ts/text/0730-semver-for-ts.md#definitions), whether a given type is intended to be constructed by users impacts its public API contract. See the discussion on that RFC for details. For this RFC, it is sufficient to note that every published `interface` or `type` alias must explicitly specify whether it is legal for users to construct it. *Implicitly*, `RouteInfo`s are only ever provided by the framework, but only implicitly. The docs should explicitly specify that the type can be imported and named, but not implemented by users.

[^transition-on-DT]: Historically, users on DefinitelyTyped have had to import `Transition` and `RouteInfo` from private locations *or* use type hacks to get them from the return types of other functions. This is because Typed Ember has consistently maintained a rule that we only publish types corresponding to the documented public API of Ember itself, and there was no possible public API for these types as *type-only* exports, since Ember was not publishing types.


### Migration docs

As discussed briefly above, existing Ember TypeScript users have used the `@types` packages from [DefinitelyTyped][DT]. We must document a migration path for users, to be published as part of the blog post announcing officially-published types for any given package. This will include both a general migration path, common to all of these, and then specific details dealing with migration of specific Ember Classic patterns.


#### General migration guide

The outline of that migration path, regardless of the specific package, is:

1. Upgrade to a version of the package (e.g. `ember-source`) which officially publishes types.
2. Remove the corresponding `@types` package(s) from your dependencies.
3. Check that your program compiles. Here, we will provide a guide of expected changes, if any.
4. Invite bug reports, directing them to the package in question!

We will *need* to publish one of these guides/posts for each package or group of packages which begins officially publishing types, at the time they begin doing so (as well as in the release notes for the package in question, of course). Users who have types from both the package itself and a `@types` package may end up confused or experiencing odd issues in their editor, so we need to make this messaging very clear!

As described above in [**Detailed design: Rollout**](#rollout), we can and should start shipping types incrementally as packages are ready, rather than gating the ecosystem on *everything* being ready. Accordingly, we will probably need a fair number of these posts!

For example, `@ember/test-helpers`, `ember-qunit`, and `ember-mocha` could begin publishing types more or less immediately after this RFC is accepted, while for `ember-source`, the publication of types would happen in line with the normal release train, as described in [**Publishing types and release channels**](#publishing-types-and-release-channels) above. Thus, we could do publish a blog post for the release of official types in the infrastructure, which would guide people to the correct version to update to, and tell them to remove the `@types/ember__test-helpers` and `@types/ember-qunit` or `@types/ember-mocha` packages from their dependencies. In that specific example, we would not expect there to be any changes between the types as published on DefinitelyTyped and those published from the packages directly.

Similarly, when a minor version of `ember-source` is published with types, we should publish a dedicated migration post, to be published at the same time as, and linked from, the announcement blog post. That post *will* include the expected changes consumers can make. However, since we *know* the expected sets of changes there (as discussed under [**Ember Classic Features**](#ember-classic-features) below), we can also guide Ember developers to start making those changes *sooner*, by publishing a post outlining the details as soon as we begin publishing the types on our canary channel. Then the announcement blog post can reiterate those as well as modifications made in response to pre-release testing.


#### Ember Classic features

Besides the general migration guide, we will also provide tips for code which uses Classic features:

- for `get` and `set`, guidance about switching to Octane idioms

- for `.extend()`, `.create()`, `.reopen()`, and `reopenClass()`, examples of using [declaration merging][declaration-merging] to correctly represent the runtime result of those types


## Drawbacks

The major drawbacks here are the same as those in [RFC #0724][rfc-0724]:

> Adding TypeScript support imposes an additional maintenance burden on all contributors to Ember:
>
> - documentation must be kept up to date in both JavaScript and TypeScript
> - changes to APIs must be kept in sync with published types (when the types are not generated from the implementation)
> - managing support for TypeScript versions will require additional effort and versioning coordination
> - presumably, API designs will be constrained in new ways (though this may also be an upside!)

There are also a few drawbacks to the specific recommendations around the rollout design here:

- Providing a more minimal support path for Classic features could make adoption more difficult for some users who have not yet fully migrated to Octane.
- Using a “rolling majors” support policy *does* impose additional upgrade cost on our TypeScript users.


## Alternatives

The major alternative here is the same as that in [RFC #0724][rfc-0724]:

> TypeScript support has historically been managed by the community. We could continue with this approach, including e.g. investigating other alternatives to DefinitelyTyped for supplying types in a more robust way. This has worked reasonably well to date, though it has added friction for adopters, especially when the Typed Ember maintainers could not keep up with changes to Ember itself.

In terms of the details of the rollout specifically, however, there are a few specific alternative approaches available to us.


### Documentation

- We could leave the TypeScript docs in a separate location entirely, as they are today. This would require many of the same costs we pay today in terms of maintenance, and would also leave TypeScript something of a second-class citizen.

- We could build an entirely separate tooling stack to support TypeScript vs. JavaScript in our documentation. This would create major maintenance overhead, and again would mirror the situation we find ourselves in today with the ember-cli-typescript docs representing a totally separate “source of truth” for that part of the ecosystem.

- We could use this as an opportunity to investigate rebuilding our documentation tooling entirely. While attractive in some ways, these projects are time-consuming and this would introduce considerable risk and likely major timeline delay. Instead, using the things we learn from building support for TypeScript can and should inform future efforts.


### Classic features

Per the edition support policy, we could choose *not* provide direct support for Ember Classic features in the core types. Since those types are still in use throughout the ecosystem, including in key ways for Ember Data, we would need to support them *somehow*, but we could do so by instead providing a *bridge* for working with them, analogous to the legacy support packages provided at major version changes.

In that design, we would ship a package like `@ember-types/classic`, which took advantage of [module augmentation](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#module-augmentation) to provide enhanced versions of those types. This would allow us to keep the supported public API published *by Ember* much smaller and simpler, while also providing an escape hatch for parts of the ecosystem which are still transitioning to Octane idioms (including especially Ember Data).

This would be a reasonable approach—and was in fact what the other Typed Ember folks and I originally thought we would do!—but we decided against it because of the problems with making the classic types work well *at all*. The approach we are *actually* proposing is minimal and easy to support, minimizes impact to the ecosystem as it stands, and actually provides *better* feedback during the transition period.


### Semantic versioning options

- Instead of adopting the “rolling window” TypeScript version policy, we could lock each Ember major version to the minimum supported version of TypeScript at the time the Ember major was released. For example, if TypeScript 5.0 were the minimum supported version for Ember 5.0, it would *remain* the supported version throughout the Ember 5.x life-cycle. The major downside here is that it prevents Ember itself from taking advantage of any new type system features for the entirety of the Ember major release.

    For context, if we had adopted the “simple majors” policy for TypeScript with the Ember 3.x era, we would have been stuck with TypeScript 2.6 or 2.7 until Ember 4.0’s release in November 2021. In that time frame, there were over 20 TypeScript releases, some of which included features which allowed for major improvements to the experience of authoring Ember apps and addons and caught many more bugs.

- Instead of allowing Ember’s published types to support only the current edition at a major, we could require that all public API be supported as long as it exists. The downside to this approach is primarily for the Ember 4.x era, where robustly supporting Ember Classic idioms with TypeScript is difficult at best and in some cases impossible. Presumably, by the time of Ember 5.0, it will be possible to publish types directly from Ember’s source (with no use of ambient types), so supporting pre-Polaris idioms may be *lower* effort than not.

    However, we fundamentally *need* that flexibility for official TypeScript support in the Ember 4.x era while we finish removing Classic APIs. Moreover, it will not hurt us to have the flexibility available for future releases: having the option on the table does not require us to use it.

- Instead of using `strict: true`, we could adopt a specific set of flags to support for the life of an Ember major series—presumably, the set of flags corresponding to `strict: true` at the time of a major version. However, this would mean we would not be providing benefits to users who *are* using `strict: true`, and might put us in the position of having to tell users *not* to use `strict: true` because our types don’t support it. **This is not merely hypothetical:** it was the case for 3–4 years for Ember’s types on DefinitelyTyped! The result is maintenance churn and needlessly lower strictness than available from TypeScript.


### Documentation

- The guides for migrating from DefinitelyTyped could be hosted as dedicated documentation rather than blog posts. This would have the upside of being long-lived documentation for apps which make this change at later points (since it requires updating to the specific Ember version before it becomes possible). However, it has the downside of being dedicated documentation we must maintain which is *only* valuable for that one specific transition, and it is really part of the "upgrade" process for apps using TypeScript.

- This RFC could specify details for documenting the disctinction between “primary” and “seconadry” packages (with implications for their SemVer commitments) in a future where the packages *do* actually exist and can be installed independently of each other. However, since it is not clear when that will happen, or what the versioning design for those packages will be, it seems more appropriate to defer any such discussion to an RFC which designs those features as a coherent whole.


## Unresolved questions

None!
