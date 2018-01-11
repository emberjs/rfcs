+ Start Date: 2017-09-07
+ RFC PR: (leave this empty)

# Summary

The goal of this RFC is to solidify the next version of the mechanism that
`ember-cli` uses to build final assets. It will allow for a more flexible build
pipeline for Ember applications. It also unlocks building experimental features
on top. It is a backward compatible change.

# Motivation

The [Packager
RFC](https://github.com/chadhietala/rfcs/blob/packager/active/0002-packager.md)
submitted by [Chad Hietala](https://github.com/chadhietala) is a little over 2
years old. A lot of things have changed since then and it requires a revision.

The current application build process merges and concatenates input broccoli
trees. This behaviour is not well documented and is a tribal knowledge. While
the simplicity of this approach is nice, it doesn't allow for extension. We can
refactor our build process and provide more flexibility when desired.

Most importantly, the approach described below helps us achieve:

+ defining and developing a common language around the subject
+ removing highly coupled code and streamline technical implementation (Ember
  Engines and Fastboot)
+ unlock a whole different set of plugins we couldn't have before:
  + ability to create custom bundles (i.e per-engine and per-route bundles)
  + take advantage of [HTTP2
    multiplexing](https://http2.github.io/faq/#why-is-http2-multiplexed) and
    [cache
    pushing](https://www.mnot.net/blog/2014/01/30/http2_expectations#4-cache-pushing)
  + optimising plugins (JavaScript and CSS tree-shaking)

# Scope

+ New public API for customising build process and giving more granular control over
the final build output

# Terminology

+ **Packaging** - The process of designing, evaluating, and producing final build assets.

# Detailed design

The detailed design is separated in various sections so that it is easier for a
reader to understand.

## Packaging

It gives you granular control over the final build output. It could be used in many
different ways (we are going to go over use cases below). Note, it isn't meant to be
used for "postprocess" transformations; "postprocess" is called after packaging is
finished.

Currently, Ember.js application and all of its depedencies get assembled under one
directory with the following structure:

```ruby
bundler:js:input/
├── addon-tree-output/
├── the-app-name-folder/
├── node_modules/
└── vendor/
```

where:

+ `addon-tree-output` is a folder that contains dependencies from Ember add-ons.
+ `the-app-name-folder` is a folder that contains Ember application code.
+ `node_modules` is a folder that contains node dependencies.
+ `tests` is a folder that contains test code.
+ `vendor` is a folder that contains other dependencies.

Note, for clarity purposes we should rename `addon-tree-output` to `addon-modules` as
both `tree` and `output` don't communicate well about the contents of the folder.

During packaging process the final output will be generated (everything that currently
resides under `dist/` folder when a developer runs `ember build`).

### `package` API

A new public `package` method will be introduced to open up a way to customise packaging process:

```javascript
// ember-cli-build.js
const EmberApp = require('ember-cli/lib/broccoli/ember-app');

module.exports = function(defaults) {
  const app = new EmberApp(defaults, {
    package(inputTree) {
      // customise `inputTree`
      // and return customised `inputTree`
    }
  });

  return app.toTree();
}
```

`package` function has the following signature:

```typescript
interface EmberApp {
  package(inputTree: BroccoliTree): BroccoliTree;
}
```

where `inputTree` will have the following structure:

```ruby
bundler:js:input/
├── addon-modules/
├── the-app-name-folder/
├── node_modules/
├── tests/
└── vendor/
```

Note, that `package` method must return a broccoli tree.

This change should be behind an experiment flag, `PACKAGING`.
This will allow us to start experimenting right away and not being
tied to a particular release cycle.

Note, that `package` is optional. If you don't define it, you're
effectively "opting out" of the feature and using the default
behaviour.

### `defaultPackager` API

It's important to make it easy for users to still use default Ember CLI
packaging.

`defaultPackager` is a way for the users to access out-of-the-box packaging while
still be able to customise the final build output.

```javascript
// ember-cli-build.js
const EmberApp = require('ember-cli/lib/broccoli/ember-app');
const defaultPackager = require('ember-cli-default-packager');

module.exports = function(defaults) {
  const app = new EmberApp(defaults, {
    package(inputTree) {
      // customise `inputTree`

      return defaultPackager(app, inputTree);
    }
  });

  return app.toTree();
}
```

`defaultPackager` has the following signature:

```typescript
function defaultPackager(app: EmberApp, inputTree: BroccoliTreel): BroccoliTree;
```

`defaultPackager` must return a `BroccoliTree`.

### Possible usages

#### Debug/Analyse

One of the applications of `package` API would be to run different analysis on the
Ember applications. Take
[broccoli-concat-analyser](https://github.com/stefanpenner/broccoli-concat-analyser),
for example. This could be easily incorporated into the build.

```javascript
// ember-cli-build.js
const EmberApp = require('ember-cli/lib/broccoli/ember-app');
const defaultPackager = require('ember-cli-default-packager');

module.exports = function(defaults) {
  const app = new EmberApp(defaults, { });

  app.package = function(inputTree) {
    const analysedTree = new BroccoliConcatAnalyser(inputTree);

    return defaultPackager(app, analysedTree);
  }

  return app.toTree();
}
```

#### Static Assets Split

One of the techniques for improving site speed is isolating changes throughout
application deployments. Assuming  the application assets are uploaded to CDN,
the reasoning is very simple: if `ember.js` or `jQuery` (possibly along with
other third party libraries) don't change with every deployment, why bust CDN
cache for them?

#### ES6 Modules

ES6 modules are [starting](https://caniuse.com/#feat=es6-module) to land in browsers.
This means that you can use `<script type="module" src="/my/app.js"></script>`.

This [article](https://philipwalton.com/articles/deploying-es2015-code-in-production-today/) [explains](https://philipwalton.com/articles/deploying-es2015-code-in-production-today/#is-this-really-worth-the-extra-effort) the benefits of using ES6 modules over ES2015 (smaller total file sizes, faster to parse and evaluate).

`package` API will make it possible to package your application for both ES2015 only browsers as well
the ones with ES6 modules support.

# Topics for Future RFCs

While working on this RFC, some ideas were brought into focus regarding existing
and new features in Ember CLI. They all likely require separate discussions in
future RFCs, but the discussion points have been included below.

## Tree-shaking

Firstly, what's _tree-shaking_? AFAIK, the term
[originated](https://groups.google.com/forum/#!msg/comp.lang.lisp/6zpZsWFFW18/-z_8hHRAIf4J)
in Lisp. The gist of the idea is "how about we start using _only_ the code that
we actually need?"

Secondly, how is it different from [Dead Code
Elimination](https://en.wikipedia.org/wiki/Dead_code_elimination)? [Rich
Harris](https://twitter.com/Rich_Harris)
[offers](https://medium.com/@Rich_Harris/tree-shaking-versus-dead-code-elimination-d3765df85c80)
a pretty good explanation in the context of [Rollup](https://rollupjs.org/). The
gist is dead code elimination happens on a final product by removing bits that
are unused. Tree-shaking is quite different - given an object we want to construct, what is the exact set of dependencies we need?.

With this RFC, we lay out the foundation and create a framework by which both
dead code elimination and tree-shaking code be implemented.

However, there are still several things that are missing:

+ **Linker** - Responsible for resolving and reducing the graph to a tree
  containing only reachable modules.
+ **File System Resolver** - Responsible for connecting a module name with a
  file path.

`Linker` would be responsible for:

+ building a minimal dependency graph as well as check for redundant edges in
  the graph (more on the topic, [Transitive reduction of a directed
  graph](https://en.wikipedia.org/wiki/Transitive_reduction#Graph_algorithms_for_transitive_reduction));
+ producing an application tree with only used modules

Dependency graph represents dependencies using module names, there is a need to
be able to convert module name to file path. This is where `File System
Resolver` comes in. Here's couple of examples:

```javascript
fileSystemResolver.resolve('lodash') => `some-path/node_modules/lodash/lodash.js`
fileSystemResolver.resolve('ember-ajax') => `some-path/addon-modules/ember-ajax/index.js`
fileSystemResolver.resolve('ember-data') => `some-path/addon-modules/modules/ember-data/index.js`
fileSystemResolver.resolve('ember-data/-private') => `some-path/addon-modules/modules/-private.js`
```

This effort could be broken down into several phases:

+ dead modules elimination inside of the `addons/` (application would be the
  main entry point and unused modules are removed only from `addons/`)
+ dead modules elimination inside of the `app/`
  + removing unused components and helpers (requires analysing templates)
  + removing unused initializers/services (this likely entails work on
    dependency injection layer as we would need access to a resolver resolution
    map)
+ tree-shaking (Rollup-like tree-shaking where we include *only* the code that is
  used)

`Linker` would be able to take an `exclude` list of modules as a parameter.
Although, valuable in some situations, it should be clearly marked as advanced
API. It should be used as a last resort and serve as an "escape hatch".

It would make sense to implement `Linker` as a strategy. Developers would be
able to "opt in"/"opt out" of optimising behaviour.

## Deprecating `app.import` API

Ember applications which choose to use `Linker` strategy should be able to
remove usages of `app.import`.

## Tools

With growing complexity of Ember applications, it is crucial to provide more
insights into final assets.

Main goals are:

- report raw/uglified/compressed asset sizes;
  [broccoli-concat-analyser](https://github.com/stefanpenner/broccoli-concat-analyser)
- find source code duplication across your javascript assets (enables you to
  fine tune code splitting parameters to reduce bundle invalidation rates as
  well as improve repeat page load performance)

# How We Teach This

This is a backward compatible change to the existing Ember CLI ecosystem. In
order to teach users how to use `package` API, we need to update the API docs
with a section for this and the best practices of when to use this. A more
general purpose blog post could be beneficial as well.

# Drawbacks

There are several potential drawbacks that are worth noting.

_Build performance_. There is minimal overhead in instantiating strategies and
calling methods on them and I believe this approach shouldn't degrade build
performance.

_A note on add-ons_. Add-ons don't rely on the way Ember CLI does bundling. That
means existing build system continues to work as expected and add-ons won't have
to change their implementation.

# Alternatives

This RFC allows us to customise packaging when needed.
[Webpack](https://webpack.js.org) has become very popular in solving this
similar problem. One could implement a `package` function that would use Webpack
for packaging. Ultimately, we need something that is aware of how Ember apps are
assembled and how Ember apps utilise dependency injection that takes advantage
of existing tools. The long term plan is to have a dependency graph that is
aware of application structure so can avoid the "wall of configuration" that
other asset packaging systems are susceptible to.

# Unresolved questions

+ Will it increase build time?
+ Should we introduce the same API on add-on level?

# Thanks

Many thanks for [@stefanpenner](https://github.com/stefanpenner/),
[@rwjblue](https://github.com/rwjblue/) and
[@chadhietala](https://github.com/chadhietala/) for helping me to drive this
forward.
