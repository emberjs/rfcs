---
Stage: Accepted
Start Date: 2021-04-27
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember CLI
RFC PR:
---

# Enable Embroider

## Summary

This RFC proposes enabling Embroider as the default build system for ember-cli.

## Motivation

The current build system provided by ember-cli (lib/ember-app.js) builds and combines your application and addon's code in a very dynamic manner. While this works, it comes with a cost of forcing additional complexities at build time while also bloating application sizes. Embroider moves away from this dynamic behavior by upgrading your addons to be statically analyzable into what is referred to as [V2 Packages](https://github.com/emberjs/rfcs/blob/master/text/0507-embroider-v2-package-format.md). Embroider can then operate on these packages to know exactly what is or is not needed to be a part of your application and can use that information to produce spec compliant JavaScript that all modern packagers (like Webpack) understand. At this point we can leverage the benefits of a particular packager such as Webpack's tree-shaking abilities and dynamic imports (to name a few).

Additional information about the motivation and key features of Embroider can be [found here](https://github.com/embroider-build/embroider/blob/master/SPEC.md).

## Detailed design

This design will cover a few topics. The first discusses the explicit file changes that happen for your application to use Embroider. Next it will cover how both new applications move to Embroider and how existing applications will migrate to it. Lastly, it will cover some deprecated concepts that no longer apply under Embroider along with how we get community addon adoption.

### Packages

The following packages will be added to package.json:

- @embroider/compat - Stage 1 is responsible for upgrading addons from their current implmentations ("V1") into a new format called "V2 Packages". This package also contains the entry point method that is invoked in your ember-cli-build.js file (shown below).
- @embroider/core - Stage 2 is responsible for taking a collection of V2 packages and abstracting away the "emberness" into standard Javascript which all packagers can understand.
- @embroider/webpack - Stage 3 of Embroider's design allows you to bring your own packager. By default Embroider ships with and recommends using Webpack

### Build File Changes:

Embroider replaces the logic provided by `ember-app.js`. By default it runs in a "safe" mode that provides the best backwards compatibility guarantees but prevents some optimizations from happening. This safe mode configuration is what we will ship by default:

```diff
-return app.toTree();
+const { Webpack } = require('@embroider/webpack');
+return require('@embroider/compat').compatBuild(app, Webpack);
```

### New application flow

The above changes will be applied to the blueprint for `ember new`. Additionally, the build flag option of `--embroider` will exist so that the option will be recorded in `config/ember-cli-update.json` like:

```json
{
  "schemaVersion": "1.0.0",
  "packages": [{
    "name": "ember-cli",
    "version": "X.X.X",
    "blueprints": [{
      "name": "app",
      "outputRepo": "https://github.com/ember-cli/ember-new-output",
      "codemodsSource": "ember-app-codemods-manifest@1",
      "isBaseBlueprint": true,
      "options": [
        "--embroider"
      ]
    }]
  }]
}
```

This will ensure that `ember-cli-update` only applies these changes to applications that are already on Embroider.

### Existing application migration flow

In order to make the migration as simple as possible, a codemod will be created. The codemod CLI will provide two commands: `preflight` and `migrate`.

Example usage:

`npx @embroider/cli preflight`
`npx @embroider/cli migrate`

#### Preflight

The goal of this command is to give users an easy yes or no answer to the question: can I start migrating to Embroider? As an example, we can check if the project is on the minimal set of package versions of known "fixed" addons along with other static checks.

#### Migrate

The migrate command is purely to codemod the file changes above to get your app onto Embroider.

### Deprecations

While Embroider strives to be as backwards compatible as possible, some existing concepts will not directly apply under Embroider. These concepts largely rely around the fact that Embroider does not want to you care about the contents of your output. This means that concepts like fingerprints asset file names or modifing output paths of your dist assets will not be supported.

#### Fingerprinting

In classic builds there was a `app.options.fingerprint.exclude` configuration which users could configure to disable fingerprinting for specific files or `app.options.fingerprint.enabled` field to disable fingerprinting for all files. Under Embroider, JS and CSS assets will always be fingerprinted in production mode (all app code is always fingerprinted even in dev mode as they will become "chunks") except for any asset that contains the data attribute `data-embroider-ignore`. An example of this is that `testem.js` is [excluded from fingerprinting](https://github.com/ember-cli/ember-cli/blob/1301d24c8d7dec1f1559cffd0688d46bcd50284e/lib/broccoli/ember-app.js#L351-L355) as that explicit file name is expected. However, under embroider we can change `tests/index.html` to contain:

```html
<script src="/testem.js" integrity="" data-embroider-ignore></script>
```

#### Output Paths

Under Embroider the output paths of `/dist` are no longer predetermined. For example the application's javascript which would normally be `/dist/assets/app_name.js` will become for example: `/dist/assets/chunk.adjasld33lk23jol.js` + `/dist/assets/chunk.zsiojladj20ljk23jl.js`. The amount of "chunks" can vary based on the size / structure of your app especially if you have route based code splits, lazy engines, and/or dynamic imports. Each of these will create their own unique chunk(s). With Embroider the outputPaths API will no longer be supported.

#### Minifying

With Embroider the responsiblity of minifing assets is a responsiblity of the 3rd stage (in the case of Webpack it is provided by the Terser Webpack plugin). Under Embroider, all production assets are minified and Terser configuration options will have to be passed as a part of [packager options](https://github.com/embroider-build/embroider#options) instead of via ember-cli-terser:

```js
var app = new EmberApp({
  'ember-cli-terser': {
    terser: {
      compress: {
        sequences: 50,
      },
      output: {
        semicolons: true,
      },
    },
  },
});
```

to

```js
+return require('@embroider/compat').compatBuild(app, Webpack, {
  webpackConfig: {
    optimization: {
      minimizer: [
        new TerserPlugin({
          terserOptions: {
            compress: {
              sequences: 30,
            },
            output: {
              semicolons: false,
            },
          },
        }),
      ],
    },
  },
});
```

### Community Adoption

Embroider relies on all addons being able to be compliant with the V2 package spec. Embroider is itself able to handle most addons by converting them to V2 but some addons are too dynamic to be automatically converted. As the community adopts Embroider, more and more community addons will need to be fully working under Embroider to ensure its successful adoption. Embroider has created a [testing package](https://github.com/embroider-build/embroider/blob/master/ADDON-AUTHOR-GUIDE.md) for addon authors to consume in their ember-try scenarios to verify their addons work under Embroider. A meta issue will be created tracking the top 100 addons according to emberobserver.com. This issue will track the adoption of @embroider/test-setup for each addon and help create a working list of which addons work under Embroider and which ones need help.

## How we teach this

We currently discuss the existing build system at: https://cli.emberjs.com/. These guides will need to be updated to ensure they reflect any changes that not longer apply to Embroider and we should add additional sections for the new functionality Embroider brings.

The following pages will need to be modified or updated:

### https://cli.emberjs.com/release/

- "What are addons?" section will need to be discuss V2 packages
- "Why do we need a CLI?" section will need to be modified as the CLI itself is no longer a "packager"
- "Contributing" section will need to reference Embroider's organization and repos

### https://cli.emberjs.com/release/basic-use/folder-layout/

- File/directory dist/'s purpose will need to be modified as <app-name>.js no longer exists

### https://cli.emberjs.com/release/advanced-use/

- The overview should contain information about Embroider

### https://cli.emberjs.com/release/advanced-use/asset-compilation/

- "Minifying" section needs to be changed
- "Fingerprinting" section needs to be changed
- "Configuring output paths" section needs to be removed

### https://cli.emberjs.com/release/advanced-use/debugging/

- Debugging tips for Embroider need to be added

## Drawbacks

The largest drawback is complexity. Embroider is effectively trying to replace the foundation of your application while keeping it standing. While the end goal is a far superior foundation that enables amazing features such as tree shaking and code splitting, this transition does require large shifts in the broader ecosystem for it to be successful.

## Alternatives

Reference to Embroider's V2 Package RFC [alternative design section](https://github.com/emberjs/rfcs/blob/master/text/0507-embroider-v2-package-format.md#alternative-designs)

## Unresolved questions

What to do with `lib/broccoli/ember-app` and related code in ember-cli? There should be a follow up RFC to deprecate the "classic" build system and its related infra.
