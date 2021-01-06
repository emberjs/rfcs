---
Stage: Accepted
Start Date: 2020-01-06
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): ember-cli
RFC PR: https://github.com/emberjs/rfcs/pull/699
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

# Codemod blueprints

## Summary

Historically, blueprints are not extensible: they exist independently and con be
overridden.
This constraint, while simple in implementation, results in difficult to maintain
blueprints that are in the _extending_ category e.g.: typescript versions of
components, routes, etc.

This RFC proposes some changes to ember-cli to support the ability to transform
any file as a part of the blueprint `generate` process. Though, the specifics of
transforming are out of scope.

## Motivation

When managing large sets of conventions internal at a company, or as part of a
community effort (such as TypeScript), blueprints have had to be duplicates of
the official or upstream blueprints, then requiring many more tests and maintenance
as copied blueprints easily get out of sync. For projects such as
[ember-cli-typescript-blueprints](https://github.com/typed-ember/ember-cli-typescript-blueprints/tree/master/blueprints),
where there are ~30 blueprints copied from ember-source, but only with additional
TypeScript notation, the maintenance burden is great -- and the community has
felt this burden as the typescript blueprints being out of date is often one of
the first things that people ask about in Discord.

Additionally, supporting "extending" of blueprints will allow companies who
generate many apps that need some custom configuration / packages / boilerplate
will be made easier to maintain via codemod than duplication. While it's true
that the app/addon blueprints don't change _that_ often, being able to
programmatically work with changes via ASTs has proven maintainable as we have
utilized codemods for other upgrade efforts within more general app/addon code.

## Detailed design

Most of the work would take place in [ember-cli/lib/tasks/generate-from-blueprint.js](https://github.com/ember-cli/ember-cli/blob/master/lib/tasks/generate-from-blueprint.js)

1. Blueprints need to be addressable via path so that an **upstream** blueprint
   may be specified. This is necessary because the **upstream** blueprint will need
   to run first.
   [PR: 9387](https://github.com/ember-cli/ember-cli/pull/9387)
   (brings `generate` up to parity with `new`)
2. Allow the `generate-from-blueprint` task to be provided options that normally
   live on the `GenerateCommand`
3. Provide fileInfos to the afterInstall hook so that an addon author can know
   what files were generated, and retrieve their generated paths and contents

The actual transforming is up to addon authors and there are several tools to
handle the transforming of various file types and they don't need to be bundled
with ember-cli.

### Example

Inspired from some of what is presently implemented in [ember-codemod-blueprint](https://github.com/NullVoxPopuli/ember-codemod-blueprint/blob/bb854e64c93912a6db96d140a3f5f0e81bebcce7/lib/base-blueprint.js)

```js
module.exports = {
  async beforeInstall() {
    // Runs all lifecycle hooks
    await this.taskFor('generate-from-blueprint').run(
      /* options for upstream blueprint */
    );
  },

  async afterInstall({ fileInfos }) {
    let { filePath, contents } = fileInfos['__root__/__path__/__name__.js'];

    // do some transform

    await fs.writeFile(filePath, newContents);
  }
}
```

## How we teach this

Initially, documentation should remain unchanged, as this should be considered
experimental and is intended for addon authors only at first.
[This repo (ember-codemod-blueprint)](https://github.com/NullVoxPopuli/ember-codemod-blueprint/pull/4)
proposes a high level API that people can experiment with (once the low level
features are implemented).

Using the above addon, an example blueprint _could_ look something like this:

```js
// <addon-root>/blueprints/<blueprint-name>/index.js
const { codemodBlueprint } = require('ember-codemod-blueprint');

module.exports = codemodBlueprint({
  upstream: 'ember-source/blueprints/component',
  description: 'My codemod transform',

  transform(options, helpers) {
    let { transformTemplate, transformScript } = helpers;
    let { project, args, fileInfos } = options;
    let projectRoot = project.root;

    // blueprint/files
    let scriptPath = fileInfos['__root__/__path__/__name__.js'];
    let templatePath = fileInfos['__root__/__template__path/__name__.hbs'];

    // using jscodeshift transforms imported from other files
    await transformScript(scriptPath, transformA, transformB, transformC);
    // or inline
    await transformScript(
      // [String] path to JS or TS file
      scriptPath,
      // [AST] root
      // [jscodeshift] j
      ({ root, j }) => {
        return root
          .find(j.Identifier)
          .forEach(path => {
            // do something with path
          });
      }
    );

    // using ember-template-recast transforms imported from other files
    await transformTemplate(templatePath, transformD, transformE, transformF);
    // on inline
    await transformTemplate(
      // [String] path to template
      componentPath,
      // [Object]
      //    ember-template-recast's transform's plugin callback
      //    https://github.com/ember-template-lint/ember-template-recast#transform
      //
      // NOTE: this transform is synchronous
      (env) => {
        let { builders: b } = env.syntax;

        return {
          MustacheStatement() {
            return b.mustache(b.path('replaced-statement'));
          }
        }
      }
    );
  },
});
```

## Drawbacks

If mutation of data is required for a transform to have an effect, that could be
tricky for folks who are used to mutation -- but return a specific shape of data
from any of the lifecycle hooks could lead to similar problems.

Codemods in general are very mutation-oriented, so I think sticking with that pattern
would be ideal.

## Alternatives

- Generate the file in memory and pass the file paths and contents in the
  beforeInstall hook and allow modification there.
  This would result in much less file I/O


## Unresolved questions

- Should the transforms happen as a part of beforeInstall?
- Should the transforms be their own hook (right before beforeInstall if that's
  used, or right after afterInstall)?
