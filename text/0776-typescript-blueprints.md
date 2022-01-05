---
Stage: Accepted
Start Date: 2021-11-11
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember CLI, Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/776
---

<!---
Directions for above:

Stage: Leave as is
Start Date: Fill in with today's date, YYYY-MM-DD
Release Date: Leave as is
Release Versions: Leave as is
Relevant Team(s): Fill this in with the [team(s)](README.md#relevant-teams) to which this RFC applies
RFC PR: Fill this in with the URL for the Proposal RFC PR
-->

# Author Built-In Blueprints in TypeScript

## Summary

In order to support both TypeScript and JavaScript as official languages in Ember, we should enable blueprints to be written in TypeScript and (optionally) transformed to JavaScript. We should also convert all of the existing built-in blueprints to TypeScript to take advantage of this new functionality and lay the groundwork for future TypeScript inclusion efforts.

## Motivation

Ember CLI's generators are a foundational part of Ember itself. They are one of the first elements of the developer experience that Ember prides itself on, and continue to be an essential part of the ecosystem by virtue of the fact that both apps and addons can generate their own custom blueprints. For many years, the `ember-cli-typescript` addon has provided its own set of TypeScript-flavored built-in blueprints that essentially act as drop-in replacement for the built-in blueprints that provide with Ember today. However, these blueprints have proven very difficult to maintain as it involves keeping them all up to date with every single change made to Ember itself, so much so that the current TypeScript blueprints have diverged to the point there is currently not a TypeScript blueprint for components at all.

As part of the larger effort to [make Typescript an officially supported language in Ember](https://github.com/emberjs/rfcs/pull/724), Ember should provide a single set of blueprints that will support both JavaScript and TypeScript. These new blueprints would be written in TypeScript, and would provide exactly the same JavaScript experience that today's users enjoy, while also providing several benefits over the current system:

1. Remove the disconnect between TypeScript and JavaScript blueprints discussed above, as the Typed Ember team would no longer be responsible for maintaining an entirely independent set of blueprints.
1. Allow for multiple versions of TypeScript blueprints to exist (since they'd belong to each version of Ember), rather than today where there is no way to tie standalone TypeScript blueprints to individual versions of Ember (e.g. there's no LTS version of the TypeScript blueprints today).
1. Enable TypeScript blueprint generation in any Ember app or addon right out of the box via a `--typescript` flag.
1. Enable addon authors to better support TypeScript users by allowing them to publish their own TypeScript blueprints without concern for also supporting JavaScript users.

Note that this does **not** mean that we should generate TypeScript by default, but rather that the blueprints themselves should _begin_ as TypeScript before actually being reified as actual code. By default, all of the Ember generators would still behave exactly as they do today: by generating JavaScript files. This RFC is primarily concerned with TypeScript as an _implementation detail_ for the existing blueprints, rather than being prescriptive about whether or not users should adopt TypeScript themselves.

## Detailed design

tl;dr: Blueprints will be handled exactly the same as before, with the one exception that any blueprint file written in TypeScript will be run through Babel's [`@babel/plugin-transform-typescript`](https://babeljs.io/docs/en/babel-plugin-transform-typescript) to strip out all the TypeScript syntax unless the user actually wants TypeScript output. Please read on for a more detailed explanation.

In the current generator system, new files are created from templates that reside in the `files` directory of the blueprint that corresponds to the generator that was invoked. When a user runs `ember generate service foo`, Ember CLI will locate the template at `blueprints/service/files/__root__/__path__/__name__.js`, load it into memory, perform a text replacement on all of the EJS-style tags (e.g. `class <%= classifiedModuleName => extends Service` becomes `class Foo extends Service`), and then writes the entire string out to a file at the correct path inside the parent app or addon. Notably, there is no evaluation of the code itself, as the entire process of blueprint generation is simple text replacement.

In the new version of Ember's blueprint system, this template would instead be named `__name__.ts`. The generator process would run identically (locate the template, read into memory, perform text replacement) up until it is actually time to _write_ the file. Before that occurs, the (now fully populated) in-memory version of the newly generated file would be run through Babel's [`@babel/plugin-transform-typescript`](https://babeljs.io/docs/en/babel-plugin-transform-typescript). This would strip the file of all TypeScript-related syntax, resulting in a completely valid JavaScript file with no evidence that it had begun as TypeScript file. In this way, we're able to ensure that the built-in generators would continue to perform as exactly as they have previously.

Furthermore, blueprints would have to explicitly opt-in to this behavior by adding a `shouldTransformTypeScript: true` flag to the blueprint's `index.js` file. This will make the automatic TypeScript transform behavior purely opt-in, thereby preventing any conflicts with existing projects that may already have their own custom TypeScript blueprints.

In addition to the per-blueprint flag, there will also be a new `isTypeScriptProject` flag added to `.ember-cli` that will allow users to mark an entire Ember app or addon as a TypeScript-first project. The presence of this flag would indicate that blueprints should output TypeScript by default, rather than JavaScript as they normally would. Flags specified in `.ember-cli` are automatically merged into the blueprint options when invoked, so this flag would be readily accessible to all blueprints. We'd also add a step to the `init` hook for `ember-cli-typescript` that would check for the presence of this flag in `.ember-cli` and warn if it's not set. In the future, we'd be able to set this flag as part of a `ember new --typescript` flag.

There are a few details worth calling out about this change to the generation process:

1. While the TypeScript files are parsed and transformed, they are _not_ type-checked in any way. The Babel plugin is concerned solely with the transformation of TypeScript syntax into JavaScript syntax and does not pay any attention to what the types actually are. This is a good thing, as it minimizes any performance cost during the generation process and we'd expect any actual type-checking to happen at compilation time in the user's app, using their own `tsconfig` rather than one we'd have to supply (and maintain).
   1. In the future, we DO also need the ability to test the actual TypeScript parts of the blueprints (i.e. do the blueprints output TS files that actually type-check?), but that functionality will be covered in a later RFC that is more concerned with enabling TypeScript support in Ember.
1. By default, the generators would behave exactly as they did before. However, we would also add a `--typescript` flag to the `generate` command that tells the generator to simply bypass the Babel transform and instead output TypeScript directly. Since TypeScript is a superset of JavaScript, we're able to easily accomodate both sets of users with a single blueprint file by starting out with the "higher-fidelity" TypeScript and down-leveling it JavaScript when needed.
1. Addon authors would be able to provide blueprints written in TypeScript that would "just work" in the same way that the built-in blueprints would, e.g. they'd get support for both JavaScript and TypeScript versions of their blueprints with no additional effort or maintenance.
1. No one is _required_ to write their blueprints in TypeScript. Any blueprint written in JavaScript would be handled in the exact same way that they are today.

To sum up, the changes would be as follows:

- Rewrite Ember's existing blueprints in TypeScript.
- Add a `--typescript` flag to `ember generate` that will force the command to output TypeScript instead of JavaScript (assuming a TypeScript blueprint exists).
- Add a `shouldTransformTypeScript` flag to the `Blueprint` base class that opts the blueprint in to the down-leveling behavior.
  - This defaults to `false`.
  - This flag exists so that blueprint authors can choose whether they want the auto-transform behavior in case there are already apps/addons that have custom blueprints they expect to always result in TypeScript files.
- Add a `isTypeScriptProject` flag to `.ember-cli` that can be used to determine whether to generate TypeScript or JavaScript by default.
  - This flag defaults to `false`.
  - If `isTypeScriptProject` is `true`, `ember g component foo` would generate a TypeScript file, otherwise it will generate a JavaScript file.
  - Users can override this default behavior by passing the `--typescript` flag (mentioned above), which will force the blueprint to output TypeScript as long as a TypeScript blueprint is available.

## How we teach this

For most users, there isn't much to teach here since their default experience with Ember's blueprints won't change at all. However, we should document the typescript-related behaviors in the [documentation for creating custom blueprints.](https://cli.emberjs.com/release/advanced-use/blueprints/) More specifically, we'd need to mention the 3 new flags being added (`shouldTransformTypeScript`, `isTypeScriptProject`, and `--typescript`) and explain their usage.

## Drawbacks

There should be little to no impact on the end-user as a result of the changes proposed in the RFC. The one drawback worth mentioning is that this change does introduce additional memory and performance overhead as a result of running the files through Babel transforms when converting from TypeScript to JavaScript.

That said, Ember (and most of the front-end world) already relies heavily on Babel, so this change would simply be extending a dependency that already exists. Furthermore, while there _is_ a cost to running a Babel transform as part of a generator, it will be extremely trivial given that generators only ever create a handful of files at a time, and a newly-generated Ember app runs _far_ more files through many more Babel transforms without issue. Finally, we'd be running the transform on the in-memory version of the file, so there'd be no additional I/O cost incurred.

## Alternatives

We could continue to maintain a wholly independent set of blueprint files, a la [`ember-cli-typescript-blueprints`](https://github.com/typed-ember/ember-cli-typescript-blueprints). However, this would leave us with the same challenges that exist today, and would also make it harder to provide official support for TypeScript in Ember itself.

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
> TBD?
