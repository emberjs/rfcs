---
Stage: Accepted
Start Date: 2021-03-11
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js, Ember Data, Ember CLI, Learning, Steering
RFC PR: https://github.com/emberjs/rfcs/pull/722

---

# Official TypeScript Support

## Summary <!-- omit in toc -->

TypeScript has become a key part of the front-end development ecosystem over the past several years, and powers many of the best developer experiences in the front-end ecosystem. Ember was a relatively early TypeScript adopter for its internals, and there is widespread usage in the ecosystem with community support, but to date Ember has not provided “out of the box” or official support for authoring apps or addons in TypeScript.

This RFC declares our intent to make TypeScript a first-class citizen of the Ember ecosystem, as a peer to JavaScript, in a way which makes the developer experience better for *all* Ember developers. It outlines the key constraints and goals for the effort, details a roadmap for accomplishing those goals, and provides the following definition of official support (from [Detailed Design: Defining Official Support](#defining-official-support)):

> Ember officially supporting TypeScript means: _**All libraries which are installed as part of the default blueprint must ship accurate and up-to-date type definitions for the current edition. These types will uphold a Semantic Versioning commitment which includes a definition of SemVer for TypeScript types as well as a specification of supported compiler versions and settings, so that TypeScript will receive the same stability commitments as the rest of Ember.**_


### Outline <!-- omit in toc -->

- [Motivation](#motivation)
- [Detailed design](#detailed-design)
  - [Defining Official Support](#defining-official-support)
  - [Constraints](#constraints)
  - [Non-Goals](#non-goals)
    - [Superseding or replacing JavaScript](#superseding-or-replacing-javascript)
    - [Typed Templates](#typed-templates)
  - [Roadmap](#roadmap)
    - [RFC Required](#rfc-required)
      - [Types](#types)
      - [Official documentation](#official-documentation)
      - [Authoring and Build](#authoring-and-build)
    - [Implementation Details](#implementation-details)
    - [Recommendations](#recommendations)
  - [Semantic Versioning and Supported TypeScript Versions](#semantic-versioning-and-supported-typescript-versions)
- [How we teach this](#how-we-teach-this)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Unresolved questions](#unresolved-questions)

## Motivation

TypeScript is a key ingredient of contemporary front-end development. Per community surveys like [the GitHub Octoverse](https://octoverse.github.com), the [Stack Overflow Developer report](https://insights.stackoverflow.com/survey/2020), and the [State of JS survey](https://2020.stateofjs.com/en-US/), TypeScript has exploded in popularity over the last half decade, and is the rare tool which has continuously grown in both usage and satisfaction over that time. While all of these surveys should be understood to be wildly unrepresentative of the broader front-end development community in various ways, they nonetheless provide some useful signal about the state of the ecosystem, especially when combined with experience reports from major companies like Microsoft (millions of lines of Typescript, all of it adopted voluntarily), Google, AirBnB, Dropbox, Bloomberg, Slack, and many others.

Additionally, the development of the language server protocol and the widespread adoption of the TypeScript language server has led to first-class support for TypeScript across many tools, from Vim to Visual Studio. This support also powers rich developer experiences for JavaScript-only developers, as it can provide smart autocomplete, inline documentation, and more even for codebases which are written solely in JavaScript. Thus, non-TypeScript and TypeScript users alike benefit when working with a project which supplies type definitions.

Ember users have been working with TypeScript for years with community support led by the Typed Ember team, but there is one critical problem which cannot be solved without making TypeScript support official, and several key ways in which official support would substantially improve the experience of both TypeScript and JavaScript developers:

- Solving the "SemVer problem" for Ember and TypeScript:

    - The TypeScript compiler does not follow semantic versioning, meaning anyone depending on it either explicitly (as a TypeScript user) or implicitly (as a consumer of the TypeScript Language Server, including for JavaScript editor support) is subject to uncontrolled breaking changes in their experience of Ember app and addon authoring. Supplying an official TypeScript version support policy would help stabilize the ecosystem, by guaranteeing accurate and up-to-date types which work with known supported versions of TypeScript, including the versions which power language server tooling for JS users.

    - *Many* JavaScript users (everyone who uses VS Code) currently implicitly depends on the unofficially maintained types from DefinitelyTyped via VS Code's [automatic type acquisition](https://code.visualstudio.com/docs/nodejs/working-with-javascript#_typings-and-automatic-type-acquisition) feature. However, DefinitelyTyped does not allow us to properly version types with the libraries they represent, especially Ember core libraries. Publishing types officially would allow us to substantially improve the stability and correctness of those types, providing a better experience for all Ember developers, and *particularly* benefiting TypeScript users, for whom there are currently no good mechanisms to insulate them from breaking changes.

- Making it *easy* for everyone to benefit from TypeScript-powered enhancements to the ecosystem. Since TS types power autocomplete, documentation, go-to-definition, etc., supplying type definitions for the whole Ember ecosystem will make all Ember developers' "developer experience" better. Additionally, TypeScript-powered tooling is our best bet for supplying richly interactive feedback in templates, for both JS and TS developers.

- Closing a key gap with other front-end frameworks: React, Vue, Svelte, and Angular all have strong support for TS, and use its tooling to improve the authoring experience for JS and TS developers. Having top-notch TypeScript support would make Ember more compelling.

For TypeScript Ember users specifically:

- Easing adoption for interested parties. Both the Typed Ember team and Ember core team members semi-regularly hear from parties interested in TypeScript adoption, but for whom the lack of official support is a concern or even a roadblack. By making it an officially-supported option, we will increase confidence for existing Ember users who are interested but worried about the current community-driven support.

- Improving coordination, for example by guaranteeing that everything from RFC completion to Edition completion includes not only docs updates but also full TypeScript support for any new features.

- A richer and better-integrated experience for TS user from the Ember CLI tools, such as the generators and blueprints, which have historically required hand-maintenance (and accordingly fallen sorely out of date).

In sum, making TypeScript an officially supported language for Ember will benefit *all* Ember users, JavaScript and TypeScript alike; it will solve many pain points for TypeScript users that cannot otherwise be addressed; and it will close a gap for Ember compared to other frameworks.


## Detailed design

The primary goal of this RFC is to add TypeScript as an officially supported language. This involves the following major elements:

- Determining whether we *should* make TypeScript a first-class language
- [Defining official support](#defining-official-support): what it does and does not entail
- Identifying the [constraints](#constraints) under which adoption can proceed
- Making explicit the [non-goals](#non-goals) of this project
- Building a [roadmap](#roadmap) for completing the effort

This RFC intentionally does *not* propose the concrete solutions for each of the problems we will need to solve to successfully adopt TypeScript. Instead, the [Roadmap](#roadmap) below lays out a series of problems which require RFCs to resolve, which we believe to represent the full set of work required to officially support the language—neither the full set of work we would like to see for TypeScript in the years ahead nor the solutions to those problems, but a definition of a good MVP.


### Defining Official Support

Ember officially supporting TypeScript means: _**All libraries which are installed as part of the default blueprint must ship accurate and up-to-date type definitions for the current edition. These types will uphold a Semantic Versioning commitment which includes a definition of SemVer for TypeScript types as well as a specification of supported compiler versions and settings, so that TypeScript will receive the same stability commitments as the rest of Ember.**_

Key implications of this commitment:

- *Not all* libraries maintained as part of "Ember Core" must ship types: only those which come in the default blueprint. (Of course, many such libraries will *choose* to ship types for the benefits they offer to consumers; this is a matter of what we *guarantee*.)

- We have considerable flexibility in opting *into* increased backwards-compatibility, but can move forward at a reasonable pace. For example, we may choose to support *additional* features beyond those for the current edition, but may also drop support for earlier editions at a major version milestone.

- The Semantic Versioning specification, including definitions of SemVer for TypeScript types and TypeScript compiler support policy, will be a key artifact of this process. Without it, we will not be able to provide the stability guarantees Ember users have come to rely on.


### Constraints

The following are our hard constraints in supporting TypeScript with Ember:

- We must not compromise our commitment to “stability without stagnation” and our strong Semantic Versioning guarantees. 
- Using TypeScript should never be mandatory for anyone who uses Ember.
- TypeScript support must never *degrade* the experience of JavaScript users, and wherever possible it should benefit both JavaScript and TypeScript developers.
- Types are published in the packages they represent, *not* in a third-party package.


### Non-Goals

The combination of constraints and ecosystem status lead us to several non-goals for this effort. A *non-goal* here means that it is not part of the initial road to making TypeScript available. 


#### Superseding or replacing JavaScript

This RFC aims to *add* a new first-class language to the Ember support matrix. However, it does *not* recommend replacing JavaScript with TypeScript. To the contrary: this is an explicit non-goal. Per the constraints described above, adding first-class TypeScript support should be a net positive for JavaScript-only Ember developers.

Additionally, this RFC does not propose changing the *default* experience from JavaScript to TypeScript even once we have full support for TypeScript.


#### Typed Templates

This roadmap RFC explicitly does not require support for “typed templates” as part of the path to adopting TypeScript as a community. TypeScript-powered integration between the template layer and the JavaScript context is a key goal for the Typed Ember team already. However, it is not a *gating feature* for Ember to officially support TypeScript. Rather: the feature can be rolled out and integrated with Ember CLI, the Ember Language Server, and other tooling whenever it is ready, decoupled from the other efforts here. If we reach a point where full support for type-aware template integration exists and is sufficiently robust, we can consider adding it to our definition of official Ember support via a dedicated RFC.

The *only* relationship those efforts have to this roadmap is that they will inform the design of the `@glimmer/component@2.x` TypeScript API (as described below under **Roadmap**).


### Roadmap

With this definition and associated design constraints in mind, the path to full TypeScript support in Ember requires RFCs to address key blockers, as well as some implementation details which do not require RFCs but are important for successfully delivering TypeScript support to our users..


#### RFC Required

Each of the following concerns must be addressed by RFCs. (Note that some RFCs might cover more than one of these points, but the use of a heading does not imply that a single RFC must cover all of the points within it.)


##### Types

- **The set of supported types.** Specifically, should the types support the whole API surface of Ember, including both Classic and Octane features, or only Octane features? Mixins, for example, are *not* part of the Octane programming model and are a major source of complexity in Ember’s current types; should they be included at all in types shipped with Ember?

- **Migration from DefinitelyTyped.** TypeScript users in the Ember community currently rely on the types in DefinitelyTyped. We need to define a transition story for them, which is closely related to the previous point. If we drop support for features currently supported on DefinitelyTyped, or make even well-motivated changes, how will users migrate successfully to them?

- **Semantic Versioning of types and TypeScript compiler version support.** We must *create a definition for semantic versioning of TypeScript types*, since there is currently no widely-used definition in the broader front-end ecosystem. (Hopefully this effort can benefit the broader TypeScript community!). Additionally, we must *define a support policy* for TypeScript versions: do we pin to a minimum type when we release a major, do something our Node support policy, or some other approach? (See also further discussion [below](#semantic-versioning-and-supported-typescript-versions).)

- **The `@glimmer/component@2.x` Type API.** The current Glimmer component TypeScript API works well, but research efforts into “typed templates” and [component documentation](https://github.com/emberjs/rfcs/pull/678) have both exposed the importance of being able to specify more “type parameters” for components than simply its args, including its root element(s) and blocks, if any. Additionally, the types should be readily extensible if other such concerns emerge in the future. For example: we might in the future want to be able to constrain what can and what cannot be “splatted” onto an element with `...attributes`; the type signature should be able to support that addition in a backwards-compatible way.

    (This may also be a good time to remove the `isDestroying` and `isDestroyed` properties and the `willDestroy` hook from the Glimmer Component API, to making a single set of breaking changes instead of multiple sets. Note, however, that official TypeScript support in Ember is not *gated* on those further changes.)


##### Official documentation

If Ember officially supports TypeScript, it is important that we support it in all of Ember’s documentation, including guides, API docs, etc. This requires considerable design effort, including thinking through how to present JavaScript and TypeScript (side by side? toggle-able? etc.), where to introduce discussion of TypeScript support in the guides, how API docs should be presented (and whether it may make sense to switch to something like API Extractor) and so on.

The official RFC templates for new features and for deprecations will also need to be updated:

- They must include a description of how changes affect TypeScript consumers.
- They will need to formally adopt TypeScript types as normative and required for new APIs, formalizing the rough pattern already common for new API designs.


##### Authoring and Build

- **How do consumers generate TypeScript apps and addons (and blueprints more generally)?** Should we support `ember new --typescript`? How should blueprints work? This will need to identify the path forward for authoring and publishing, Ember CLI integration, and in particular integration with the Embroider v2 Package Format. (If possible, TypeScript integration should piggy-back on the Embroider v2 *Authoring* Format, but if that format is not available, this is not a blocker.)

- **Supported compilation modes.** What should the out-of-the-box settings be for compiling Ember apps and addons written in TypeScript? This will need to be written with an eye to Embroider, preferably with the v2 Package Format as the *only* supported publication format. It must also address the transition path from current ember-cli-typescript (if any). It *must* address interoperation hazards between Babel and TypeScript: for example, in the decorators implementations, use of the `isolatedModules` setting and associated lints, etc.

- **Migration from the status quo.**  Today, users install `ember-cli-typescript` to opt into using TypeScript, and there is a wholly separate (and, unfortunately, very poorly-maintained) set of blueprints for working with TypeScript.  We will need to identify a migration story from the *current* paired design of `ember-cli-babel` and `ember-cli-typescript` to the future approach suggested by the previous bullet.


#### Implementation Details

There are also a number of key implementation concerns which must be addressed, but which, as implementation details, do *not* require RFCs:

- **How are the published types generated?** While much of Ember is implemented in TypeScript, its internal types are not currently ready to be used for public API, and it will likely take some time for them to be ready (with different packages ready at different times). We will need to determine how to publish hand-authored type-definition files, implementation-derived definition files, and over time a mix of the two until we hopefully are able to publish *only* implementation-derived files.

- **Where do published type definition files live in the artifacts published to npm?** TypeScript generally assumes that types are published with packages of the same name. While there is [now](https://emberjs.github.io/rfcs/0706-deprecate-ember-global.html) a path to publishing actual packages like `"@ember/object"`, this effort should not be gated on any such effort, and there is additional work to be done. However, that work simply consists of implementation details, rather than defining new public API.

- **How do app and addon authors *consume* published types?** There are a few “gotchas” about how TypeScript supports features like autocomplete in a project, which primarily affect the very first few interactions end users have with TypeScript, which we should address via tooling. How this works is closely related to where the type definition files are generated during publication, and so will likely need to be solved in conjunction with that issue.


#### Recommendations

Since the authoring and support policies we adopt for Ember itself will likely be adopted in an _ad hoc_ way, it may be useful to provide official *recommendations* for authors. These might include:

- Semantic Versioning guidelines
- tooling recommendations (in line with the defaults for the build pipeline)
- documentation guidelines

The existence of these guidelines is not a requirement for Ember's official adoption of TypeScript, but the adoption process would be a prime opportunity to generate these artifacts.


### Semantic Versioning and Supported TypeScript Versions

Ember and TypeScript have fundamentally different views on Semantic Versioning (SemVer).

Ember has a deep commitment to minimizing breaking changes in general, and to strictly following SemVer when breaking changes *are* made. (The use of lockstep versioning for Ember Data and Ember CLI complicates this commitment to a substantial degree, but that complication is outside the scope of this RFC.)

TypeScript explicitly *does not* follow SemVer. TypeScript's core team argues that *every* change to the compiler is a breaking change, and that SemVer is therefore meaningless. (We do not agree with this characterization, but are also uninterested in arguing it. This RFC takes the TypeScript team's position as a given.) Accordingly, every TypeScript point release may be a breaking change, and "major" numbers for releases signify nothing beyond having reached `x.9` in the previous cycle.

For TypeScript to be a first-class citizen of the Ember ecosystem, we need:

- a policy defining what constitutes a breaking change for consumers of a library which publishes types, including Ember’s core libraries
- tooling to detect breaking changes in types—whether from refactors, or from new TypeScript releases—and to minimize the amount of churn from breaking changes in TypeScript
- a general and widely-adopted policy for supported TypeScript versions

Once all three of those elements are adopted, end users will be able to have equally high confidence in the stability of published types as they do in their runtime code.


## How we teach this

This RFC intentionally defers most of the analysis of teaching work to an RFC dedicated to the question. At a high level, once the roadmap is complete, the Ember blog should announce support, and Ember’s docs should explicitly state that TypeScript is officially supported and show TypeScript examples alongside JavaScript examples.

It is critical that these materials (including the blog post) emphasize that TypeScript support is *additive*, not a replacement for JavaScript, as discussed above in [Non-Goals](#non-goals).


## Drawbacks

Adding TypeScript support imposes an additional maintenance burden on all contributors to Ember:

- documentation must be kept up to date in both JavaScript and TypeScript
- changes to APIs must be kept in sync with published types (when the types are not generated from the implementation)
- managing support for TypeScript versions will require additional effort and versioning coordination
- presumably, API designs will be constrained in new ways (though this may also be an upside!)


## Alternatives

TypeScript support has historically been managed by the community. We could continue with this approach, including e.g. investigating other alternatives to DefinitelyTyped for supplying types in a more robust way. This has worked reasonably well to date, though it has added friction for adopters, especially when the Typed Ember maintainers could not keep up with changes to Ember itself.


## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
TBD?
