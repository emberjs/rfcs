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

+ New public API for registering strategies

# Terminology

+ **Strategy** - A strategy represents a transformation. Speaking in Broccoli
  terms, it returns a transformed tree given an input tree.

# Detailed design

The detailed design is separated in various sections so that it is easier for a
reader to understand.

## Strategies

Strategy is an extensibility primitive. It gives you granular control over the
final output. It could be used in many different ways (we are going to go over
use cases below). Note, they aren't meant to be used for "postprocess"
transformations. In fact, they are solutions to different problems. The mental
model here would be: "postprocess" is called after `strategies` have been
processed.

Strategy must have the following interface:

```typescript
interface Strategy {
  constructor(options: any);
  toTree(inputTree: BroccoliTree): BroccoliTree;
}
```

`inputTree` will have the following structure:

```ruby
bundler:js:input/
├── addon-modules/
├── the-app-name-folder/
├── node_modules/
└── vendor/
```

where:

+ `addon-modules` is a folder that contains dependencies from Ember add-ons
+ `the-app-name-folder` is a folder that contains Ember application code
+ `node_modules` is a folder that contains node dependencies
+ `vendor` is a folder that contains other dependencies.

Note, that `toTree` method must return a broccoli tree.

Strategies get access to the same input tree, they are not chained off of each
other and they don't run in parallel.


### `app.import` API

`app.import` API has limited application which allows you to import and
rename files as well as specify a transformation to be applied (AMD).

One could easily implement the same behaviour with strategies. In fact,
strategies give you much more control over whole application tree, not
just separate files.

### Concatenation Strategy

It is used by Ember CLI privately in the latest `canary` version to produce
`assets/app-name.js` and `assets/vendor.js`. It takes a broccoli tree and
returns a new concatenated broccoli tree.

```javascript
const concat = require('broccoli-concat');

class ConcatenationStrategy {
  constructor(options) {
    this.options = options;
  }

  toTree(inputTree) {
    return concat(inputTree, this.options);
  }
}
```

Where `this.options` mimics [`broccoli-concat`
options](https://github.com/broccolijs/broccoli-concat#advanced-usage).

### Example of possible strategies

Below are some of the strategies that could be implemented.

#### Assets Split Strategy

One of the techniques for improving site speed is isolating changes throughout
application deployments. Assuming  the application assets are uploaded to CDN,
the reasoning is very simple: if `ember.js` or `jQuery` (possibly along with
other third party libraries) don't change with every deployment, why bust CDN
cache for them?

This strategy could be provided out of the box by Ember CLI as a part of site
speed improvement story.

```javascript
class AssetsSplitStrategy {
  constructor(options) {
    this.options = options;
  }

  toTree(inputTree) {
    return concat(inputTree, {
      headerFiles: [
        jQueryPath,
        emberPath
      ],
      outputFile: 'vendor-static.js'
    });
  }
}
```

#### Debug/Analyse Strategy

Yet another application of strategies would be to run different analysis on the
Ember applications. Take
[broccoli-concat-analyser](https://github.com/stefanpenner/broccoli-concat-analyser),
for example. This could be a strategy and easily incorporated into the build.

#### Engine Strategy

Introducing `Strategy` as a concept allows us to re-think the way we approach
Ember Engines as well. We can start thinking about engines as a strategy of
certain type. In fact, it's very similar to `ConcatenationStrategy` described
above.

+ reduces the API surface (as opposed to `ember-engines` and
  `ember-asset-loader`, only one strategy)
+ makes things explicit, meaning that you have to register a strategy

## Registering custom strategies

Ember applications would be able to register custom strategies via
`ember-cli-build.js` file by passing a `strategies` property to the constructor.

Passing `strategies` as an option to the constructor will clobber default
strategies that Ember CLI provides out of the box (they mimic existing behaviour
of creating `dist` folder with final assets).

Ember CLI will provide a way to use default strategies that people can
take advantage of.

For example:

```javascript
// ember-cli-build.js
const {
  createDefaultApplicationStrategy,
  createDefaultVendorStrategy
} = require('ember-cli/lib/broccoli/strategies');
const EmberApp = require('ember-cli/lib/broccoli/ember-app');

// no operation strategy that puts app/add-ons tree inside of
// `noop-output` folder
class Noop {
  constructor(options) {
    this.options = options;
  }

  toTree(inputTree) {
    return inputTree;
  }
}

module.exports = function(defaults) {
  const app = new EmberApp(defaults, { });

  app.strategies = [
    new Noop(),
    createDefaultApplicationStrategy(app),
    ...createDefaultVendorStrategy(app)
  ];

  return app.toTree();
}
```

Note, that `strategies` is an optional property. If you don't pass it to the
constructor or `strategies` is an empty array, you're effectively "opting out"
of the feature and using the default behaviour.

Another thing to note is that the order of strategies matter. They will
be processed in the order they are received.

This change should be behind an experiment flag, `STRATEGIES`. This will allow
us to start experimenting with different strategies right away and not being
tied to a particular release cycle.

There might be a case in the future when two different strategies need to
communicate with each other to work properly. For example, we want to
make sure strategies don't clobber each other if they happen to modify
the same files or if one strategy removes a file and the other one still
operates on it.

As opposed to implementing a message passing mechanism (which would be
rather complex), we should allow for "an escape hatch" to implement the
needed behaviour.

```javascript
const EmberApp = require('ember-cli/lib/broccoli/ember-app');

module.exports = function(defaults) {
  const app = new EmberApp(defaults, { });

  app.createOverrideStrategy = function(looseModulesTree) {
    // `looseModulesTree` is a broccoli tree with application and add-on
    // files in non-transpiled state
  }

  return app.toTree();
}
```

### Note on add-ons

Strategies "registration" are manual and explicit, not as "magical" as
Ember CLI add-ons. In the future, we want to optimise for the patterns that
emerge in the community and make it more ergonomic.

If the add-on wants to expose a strategy, it could be done via exposing
it through Node.js `require`.

```javascript
const {
  createDefaultApplicationStrategy,
  createDefaultVendorStrategy
} = require('ember-cli/lib/broccoli/strategies');
const EmberApp = require('ember-cli/lib/broccoli/ember-app');
const FastbootStrategy = require('ember-fastboot/strategies/fastboot');

module.exports = function(defaults) {
  const app = new EmberApp(defaults, { });

  app.strategies = [
    new FastbootStrategy(),
    createDefaultApplicationStrategy(app),
    ...createDefaultVendorStrategy(app)

  ];
  return app.toTree();
}
```

Final API will be fleshed out later.

#  Topics for Future RFCs

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

This is a backward compatible change to the existing Ember CLI eco system. In
order to teach users how to use strategies, we need to update the API docs with
a section for this and the best practices of when to use this. A more general
purpose blog post could be beneficial as well.

# Drawbacks

There are several potential drawbacks that are worth noting.

_Build performance_. There is minimal overhead in instantiating strategies and
calling methods on them and I believe this approach shouldn't degrade build
performance.

_Coupling with Broccoli_. The idea of having a more generic cli was discussed
during last Ember CLI core face to face meeting and it was agreed that it was a
sizeable amount of work. That kind of refactoring CLI would require a lot of
code changes and is a large body of work. At the moment, Broccoli is a de facto
Ember CLI build tool. We don't introduce any extra coupling, not more than it is
right now.

_A note on add-ons_. Add-ons don't rely on the way Ember CLI does bundling. That
means existing build system continues to work as expected and add-ons won't have
to change their implementation.

_A note on non-javascript bundles_. Strategies could be used to create
non-javascript bundles as well. For example, we could create a separate bundle
for Handlebars templates if need be. I'm going to leave technical details out as
  they are out of the scope of this RFC.

# Alternatives

_How is `Strategy` different from `BroccoliPlugin`?_ The underlying primitive is
still be a broccoli plugin (it has nothing to do with `strategies`, that's how
broccoli works). `Strategies` will likely end up being wrappers around broccoli
plugins. Going down the "everything is a broccoli plugin" path would lead to
even further coupling of Ember CLI and Broccoli which actually doesn't have to
be so. `Strategy` is more of a build pipeline primitive that can hide the build
tool completely; "register a transform and it will happen" kind of thing. That
way we could theoretically abstract "common" parts of CLI and make it less ember
specific but keep interfaces between components in place.

This RFC allows us to introduce _any_ bundling strategy when needed.
[Webpack](https://webpack.js.org) has become very popular in solving this
similar problem. We could implement a `WebpackStrategy` that would take care of
the bundling. Ultimately, we need something that is aware of how Ember apps are
assembled and how Ember apps utilize dependency injection that takes advantage
of existing tools. The long term plan is to have a dependency graph that is
aware of application structure so can avoid the "wall of configuration" that
other asset packaging systems are susceptible to.

# Unresolved questions

+ Will there be an ordering problem (strategies)?
+ Will it increase build time?
+ Which strategies are going to be supported out-of-the-box?
+ Should we introduce the same API on add-on level?
+ Will simple extension strategy “just work”?
+ Communication between strategies
+ Debugging strategies and output validation

# Thanks

Many thanks for [@stefanpenner](https://github.com/stefanpenner/),
[@rwjblue](https://github.com/rwjblue/) and
[@chadhietala](https://github.com/chadhietala/) for helping me to drive this
forward.
