- Start Date: 2020-01-10
- Relevant Team(s): Ember CLI
- RFC PR: https://github.com/emberjs/rfcs/pull/578
- Tracking:

# Ability to hook into build process without addons

## Summary

This RFC proposes adding the `treeFor<Type>` hooks to the options passed to the `EmberApp` class.
These hooks would run during the build process and allow developers to affect the build
pipelines for each of the trees (app, vendor, public, etc).

## Motivation

Currently, the recommended way to affect the Broccoli tree during build time is by hooking into
the `treeFor*` APIs provided by addons. Although it is technically possible to [merge additional
trees by passing in more trees into the `toTree()` method][1] of `EmberApp` in `ember-cli-build.js`,
this feature is not documented well and it's not clear if the core team would like to see it used.

Developers that want to use the `treeFor` APIs have to develope an addon or in-repo addon.
This is problematic for a few reasons:

- An addon is a full npm package. Even if it’s an in-repo addon, it comes with its own name and
dependency tree. The burden of maintaining another package to write a few lines
of code perpetuates the perception that Ember is "bloated" and has a steep learning curve.
- Writing code in an addon (or `lib/`) requires relying Ember CLI to invoke the code. There is no
guarantee of when or how the code will be called, so it keeps developers locked into the Ember
ecosystem. This feeling of being trapped in a build system means under-investment,
or worse: moving away.
- The entry point of the Ember build is the `ember-cli-build.js` file. The full power of a good
program should always be available in the entry point of the program. Being able to add modules
into the program should be treated as a progressive enhancement rather than a requirement.
- Although any number of addons can be added into the build process and any number of them could
affect the build process, there is no good way to guarantee the order of these addons. With hooks
exposed in a single place, it would be simple to execute code in the desired order.
- Any framework-agnostic business logic, configuration, or data needed by the build pipeline
must first be deployed independently as a module and then wrapped again in an ember-specific addon. This is a maintenance nightmare.

## Detailed design

The high level of Ember application build looks like this:

```js
const app = new EmberApp(defaults, {});
return app.toTree();
```

The `toTree()` method gathers the trees of each type from each addon and merges them together [like this][3]:

```js
const trees = [
  this.getAddonTemplates(),
  this.getStyles(),
  this.getTests(),
  this.getExternalTree(),
  this.getPublic(),
  this.getAppJavascript(this._isPackageHookSupplied),
].filter(Boolean);

mergeTrees(trees, {
  overwrite: true,
  annotation: 'Full Application',
});
```

Each of the individual `trees` (template, style, etc) gathers both explicit and implicit trees
from each addon. Explicit trees are the return value of the `treeFor()` and `treeFor<Type>` hooks
in the index.js file, whereas implicit trees are the contents of the correlated directories in the
addon (e.g. the `public/` directory for the public tree). Currently, the [order that these three sets
of trees][4] are gathered is this:

1. Explicit tree from `treeFor` hook
1. Implicit tree from directory
1. Explicit tree from `treeFor<Type>` hook

The `treeFor<Type>` hook receives the implicit tree as an argument.

This RFC proposes that after Ember CLI has merged all addon and app trees for each type (but before
it calls `postProcessTree` hooks from addons), it call `treeFor<Type>` on the `app` instance and
passes the merged tree of that type as an argument.

The end user experience would look roughly like this:

```js
const writeFile = require('broccoli-file-creator');
const path = require('path')
const fs = require('fs');

const app = new EmberApp(defaults, {
  /**
   * @param tree The merged tree from the app and all addons
   */
  treeForPublic(tree) {
    const dataPath = path.join(tree.outputPath, 'data.json');
    if (fs.existsSync(dataPath)) {
      return writeFile('data.json', JSON.stringify({ foo: 'bar' }));
    }
  },

  treeForApp(/* tree */) {},
  treeFor(/* tree */) {},
  // ...etc
});

return app.toTree();
```

## How we teach this

The Ember CLI documentation currently does not introduce Ember CLI as a build pipeline. Rather,
it explains Ember CLI as an ecosystem of addons, and covers common use cases such as asset compilation
and minification. This can make it challenging to dive into the build pipeline, when it is necessary.

Teaching these `treeFor` hooks would require first making the decision to expose the build pipeline
as a first-class citizen of the Ember CLI ecosystem.

## Drawbacks

None.

## Alternatives

Wait for Embroider and see what it offers?

## Unresolved questions

None

[1]: https://github.com/ember-cli/ember-cli/blob/v3.15.1/lib/broccoli/ember-app.js#L1791-L1799
[2]: https://github.com/ember-cli/ember-cli/blob/v3.18.0/lib/broccoli/ember-app.js#L697-L711
[3]: https://github.com/ember-cli/ember-cli/blob/v3.18.0/lib/broccoli/ember-app.js#L1646-L1649
[4]: https://github.com/ember-cli/ember-cli/blob/v3.18.0/lib/models/addon.js#L627-L632
