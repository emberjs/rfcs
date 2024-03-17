---
stage: accepted
start-date: 2024-03-17
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - learning
  - steering
  - typescript
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

<!--- 
Directions for above: 

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

# Ember Language Tooling Roadmap

## Summary

This RFC provides a roadmap towards modernizing, unifying, and improving the Ember language tooling ecosystem. In particular, we recommend the following long term goals:

1. Remove support of and any reference to GlimmerX from Glint
2. Convert Glint from yarn to pnpm
3. Convert Glint to be backed by Volar 2.0, a framework for building language tooling that's used by Vue, MDX, Astro, and more.
4. Migrate Glint to provide editor completions, type-checking, etc, using a TS Plugin
5. Merge Glint and Ember Language Server together into a singular "Ember Language Tooling" package (TBD which one gets merged into the other)
6. (Depending on #5) Move Glint to the "ember-tooling" Github Organization.
7. Strongly encourage the deprecation and removal of sprawling editor extensions with overlapping functionality in the ecosystem
8. Tie-break any debates over how to accomplish the above in favor of whatever produces the least friction with Volar.

## Motivation

### Motivation: Issues in existing Ember Language Tooling / Glint

- The language tooling experience with modern Ember's `.gts`/`.gjs` files is still not great
  - e.g. we only recently implemented support for [Auto-Import in .gts/.gjs files](https://github.com/typed-ember/glint/pull/677)
  - There is no great solution when working with repos with both .ts/.js and .gts/.gjs files.
    - Either you use "takeover" mode (disabling vanilla TS language server, using only a non-feature-complete Glint), or you run both TS and Glint servers, which has performance issues and awkward duplication of certain kinds of errors, completions, hints, etc.
- Fixing these issues will be a long slog
  - It's not fun work
  - It's error-prone, whack-a-mole guesswork to correctly implement all the necessary hooks the TS service requires in order to provide quality completions
- Fortunately, a lot of this ugly work has already been done within [Volar, the Embedded Language Tooling Framework](https://volarjs.dev/)
  - Volar is maintained by a number of folk, including a Vue.js core team member
  - The official Vue Language Tooling package is written using Volar
  - The Vue community is huge -- Vue Language Tooling and Volar are under very active (and sponsored) development.
    - We have an opportunity to hitch our wagon, so to speak, to another community that has a lot more energy to work on language tooling than the Ember community does, speaking honestly.

### Motivation: Positive results in Volar spike

As part of a [spike on converting Glint to use Volar](https://github.com/machty/glint/tree/volar-spike), I was able to validate that Volar is an excellent choice for us to shift our tooling towards.

In particular:

I was able to get the same Auto-Import functionality working in .gts files as in my [Glint PR](https://github.com/typed-ember/glint/pull/677) without writing any ugly TS service interface code. In fact, all I had to do was properly express `.gts` files as a VirtualCode (a core Volar primitive), and suddenly I got a lot of functionality for free, including auto-complete, but also a number of other commands and common Refactors:

1[](https://machty.s3.amazonaws.com/ember/glint/glint_volar_compare2.png)

One of the commands was an Organize Imports command that Glint would need [custom work](https://github.com/typed-ember/glint/pull/669) in order to implement:

1[](https://machty.s3.amazonaws.com/ember/glint/organize_imports.png)

Lastly, Volar provides a VSCode extension for debugging Volar LSPs called Volar Labs, and just by using proper Volar primitives of expressing `.gts` files as a VirtualCode that embeddeds other VirtualCodes (i.e. `<template>` tags containing Handlebars code), I was able to get Glint .gts files showing up correctly in Volar Labs convenient mapping UI (see screenshot below). This is another thing Volar gives you "for free" that we'd otherwise need Glint-specific tooling to provide (in Glint's case, a similar custom built VSCode command called "Show IR for Debugging" has been implemented with a subset of the functionality Volar Labs provides in the screenshot below):

![](https://machty.s3.amazonaws.com/ember/glint/glint_volar_labs.png)

### Motivation: Sponsorship Opportunities

There are talented folk in the Volar community that have both interest and availability in doing sponsored work to get Glint working atop Volar. The details remain to be worked out here, but it is likely that something similar to the Embroider Initiative could be put in place to fund work on improving Ember Language Tooling by building atop Volar (and that interest would not be there if we chose some other alternative).

## Detailed design

### Proposal 1: Remove GlimmerX from Glint

[GlimmerX](https://github.com/glimmerjs/glimmer-experimental) is "a set of experimental high-level APIs built on Glimmer.js". It does not appear to be under active development (last commit was 2 years ago).

There is a large chunk of the Glint codebase and documentation dedicated to GlimmerX, and for some of the changes this RFC proposes, it would be a large cost with not much value to continue to support GlimmerX.

There is another reason GlimmerX is less practical to maintain: at the time of writing it appears to use a particular flavor/implemention of Template Imports that is 1. different from Ember's .gts solution and, more importantly, 2. is incompatible with this RFC's proposal to migrate Glint's editor tooling to use a TS Plugin rather than a Language Server.

In particular, the GlimmerX way of writing templates with Template Imports is:

1. Write your components in `.ts` files (which by default triggers vanilla TS language server in most editors)
2. Use `hbs`-tagged template literals to defining templates. Any variables/references within the template close over any import statements and variables declared in the file.

The problem with this approach is that:

1. When Vanilla TS runs, it'll produce errors/warnings such as "unused reference" for all the variables/imports that are only referenced within the `hbs` template, because it has no way at this point of knowing about GlimmerX's special Template Imports syntax/behavior.
2. TS Plugins can only augment the set of errors within a file, they can't remove errors already added by default TS (or other plugins)
3. Volar is moving away from "takeover mode" language servers to TS Plugins (which has many benefits described below), and there's no way to bring GlimmerX along for the ride

Ember avoids these issues by way of a novel `.gts` / `.gjs` file extension which does not trigger vanilla TS tooling. This will allow us to implement a TS Plugin to process the file "from scratch" so that there are no default TS errors / warnings to remove.

In conclusion, most people don't even know what GlimmerX is, it's not under active development, it doesn't play nicely with our plans to use TS Plugins. Therefore, we should remove it from Glint before proceeding further with the proposals in this RFC.

### Proposal 2: Convert Glint from yarn to pnpm

This is a small chore and will unify Glint with most modern repos in the Ember community as well as Volar's repos.

### Proposal 3: Convert Glint to be backed by Volar 2.0, a framework for building language tooling that's used by Vue, MDX, Astro, and more.

This is a major milestone and where the bulk of the work lies.

Some of this work has been done in my [spike](https://github.com/machty/glint/tree/volar-spike), but most of that work is focused on .gts templates (no attempt has been made to get class-backed components + separate .hbs templates working -- see below for a discussion as to whether we should pursue that).

For this milestone, we will simply attempt to achieve feature parity with existing Glint (but likely we will get a lot of commands/refactors/other functionality for free just by moving to Volar). In particular, this milestone continue to run Glint as a Language Server (rather than TS Plugin).

### Proposal 4: Migrate Glint to provide editor completions, type-checking, etc, using a TS Plugin

Vue Language Tooling has deprecated their Language-Server-backed language tooling in favor of a TS Plugin -driven tooling (while continuing to maintain vue-tsc for actually compilation of files).

There are many benefits to shifting towards a TS Plugin approach:

1. Other non-Ember/Glint-specific TS Plugins can be used in conjunction with Glint's future TS Plugin.
  - i.e. by expressing .gts files as TS Plugin, we could use plugins/libraries like:
    - esm.sh (https://twitter.com/jexia_/status/1733032441521308014)
    - TSL (https://github.com/johnsoncodehk/typescript-linter)
2. In general, TS Plugins allow custom file types to be recognizable to other custom file types that are supported by TS Plugins
  - e.g a TS Plugin for importing xstate state machines, or .yml files, can work with .gts files, and if some other language wanted to import a `.gts` file, Glint's `.gts` file TS Plugin would facilitate that in a manner that the LSP approach would not.
3. Somewhat anecdotally, TS Plugins seem to offer better performance than language servers even though they're internally performing the same work

### Proposal 5: Merge Glint and Ember Language Server together into a singular "Ember Language Tooling" package

This is another major milestone and perhaps worth another RFC, but as it stands, notwithstanding the effort involved to merge one into the other, there is no benefit and only confusion in Ember developers having to install something called "Ember Language Server" AND "Glint" to provide them modern Ember.js language tooling, so ideally these should be merged into one.

I will leave this part underspecified for now -- it'll be easier to elaborate on what needs to happen here after we've completed our work to re-work Glint atop of Volar.

### Proposal 6: Move Glint to the "ember-tooling" Github Organization.

Regardless of how Proposal 5 is resolved, we should consolidate Ember tooling into the "ember-tooling" organization, rather than having things sprawled between "ember-tooling" and "typed-ember".

### Proposal 7: Strongly encourage the deprecation and removal of sprawling editor extensions with overlapping functionality in the ecosystem

With a unified Glint + ELS, we encourage maintainers of extensions providing Ember syntaxes / tooling to deprecate and ultimately remove their extensions from the VSCode marketplace and archive their Github repos, directing their traffic to the Ember Language Tooling package.

## How we teach this

(This RFC section not applicable.)

## Drawbacks

There is some risk to placing such a heavy bet on Volar in case they take things in a direction that is not advisable or incompatible/inconvenient to Ember. If Volar takes things in an unseemly direction, we would have to use and maintain a fork of Volar.

## Alternatives

The alternative is to continue to add features to Glint to achieve feature parity with vanilla TS, but ultimately this would mean duplicating the work of Volar and any other language tooling that isn't dependent on Volar, while probably doing a worse job implementing these features in the process.

Also, "alternatively" we could still pursue merging ELS + Glint and punt on the Volar refactor, which would still fix some points of confusion and complexity in Ember's tooling ecosystem.

## Unresolved questions

### Support class-backed components?

Continuing to support the classic two-file approach to Ember components (optional .js/.ts component file backing the .hbs template) will be more difficult than only supporting .gjs/.gts files (in fact one of the motivators for the .gts format was to avoid the complexity/ambiguity of the multi-file approach). I have not reached the point of the Volar spike to know how practical/possible it is to represent the classic two file approach.

There have been some discussions in the community as to whether we should just cut ties with the two-file past, but given that language tooling for .gts/.gjs is not that great, there's been less motivation for people (including this RFC's author) to migrate their components to .gts until tooling "is ready", and even when it is ready, it's still a lot of work for companies to migrate their code to .gts.
