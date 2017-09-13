+ Start Date: 2017-09-07
+ RFC PR: (leave this empty)

# Summary

The goal of this RFC is to solidify the next version of the mechanism that `ember-cli` uses to build final assets. It will allow for a more flexible build pipeline for Ember applications. It also unlocks building experimental features on top. It is a backward compatible change.

# Motivation

The [Packager RFC](https://github.com/chadhietala/rfcs/blob/packager/active/0002-packager.md) submitted by [Chad Hietala](https://github.com/chadhietala) is a little over 2 years old. A lot of things have changed since then and it requires a revision.

The current application build process merges and concatenates input broccoli trees. This behaviour is not well documented and is a tribal knowledge. While the simplicity of this approach is nice, it doesn't allow for extension. We can refactor our build process and provide more flexibility when desired.

Most importantly, the approach described below helps us achieve:

+ defining and developing a common language around the subject
+ removing highly coupled code and streamline technical implementation (Ember Engines and Fastboot)
+ unlock a whole different set of plugins we couldn't have before:
  + ability to create custom bundles (i.e per-engine and per-route bundles)
  + take advantage of [HTTP2 multiplexing](https://http2.github.io/faq/#why-is-http2-multiplexed) and [cache pushing](https://www.mnot.net/blog/2014/01/30/http2_expectations#4-cache-pushing)
  + optimising plugins (Javascript and CSS treeshaking)

# Scope

+ New public API for registering strategies

# Terminology

+ **Strategy** - A strategy is responsible for returning a build pipeline that can emit a specific set of output assets given an input tree.
+ **Assembler** - Responsible for taking the resolved tree and applying – default or user provided – strategies to the tree.

# Detailed design

The detailed design is separated in various sections so that it is easier for a reader to understand.

## Assembler

`Assembler` is similar to a car assembly line - starting with a metal frame, making changes to it along the way, adding parts and at the end, there's an assembled car.

It does not make any assumptions about how the final output should be constructed. Instead, it relies on strategies to tell it what the final output should look like. This is rather important as it enforces clear separation of concerns.

Assembler must have the following interface:

```typescript
interface Assembler {
  constructor(inputTree: BroccoliTree, strategies: Strategy[]);
  toTree(): BroccoliTree;
}
```

`toTree` method would gather all the trees from the passed in `strategies`, and merge them into one final broccoli tree.

## Strategies

Strategy is an extensibility primitive. It gives you granular control over the final output. It could be used in many different ways (we are going to go over use cases below).

Strategy must have the following interface:

```typescript
interface Strategy {
  constructor(options: any);
  toTree(assember: Assembler, inputTree: BroccoliTree): BroccoliTree;
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

Note, that  `toTree` method must return a broccoli tree.

### Concatenation Strategy

It is used by Ember CLI privately in the latest `canary` version to produce `assets/app-name.js` and `assets/vendor.js`. It takes a broccoli tree and returns a new concatenated broccoli tree.

```javascript
const concat = require('broccoli-concat');

class ConcatenationStrategy {
  constructor(options) {
    this.options = options;
  }

  toTree(assembler, inputTree) {
    return concat(inputTree, this.options);
  }
}
```

Where `this.options` mimics [`broccoli-concat` options](https://github.com/broccolijs/broccoli-concat#advanced-usage).

### Example of possible strategies

Below are some of the strategies that could be implemented.

#### Assets Split Strategy

One of the techniques for improving site speed is isolating changes throughout application deployments. Assuming  the application assets are uploaded to CDN, the reasoning is very simple: if `ember.js` or `jQuery` (possibly along with other third party libraries) don't change with every deployment, why bust CDN cache for them?

This strategy could be provided out of the box by Ember CLI as a part of site speed improvement story.

```javascript
class AssetsSplitStrategy {
  constructor(options) {
    this.options = options;
  }

  toTree(assembler, inputTree) {
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

Yet another application of strategies would be to run different analysis on the Ember applications. Take [broccoli-concat-analyser](https://github.com/stefanpenner/broccoli-concat-analyser), for example. This could be a strategy and easily encorporated into the build.

#### Engine Strategy

Introducing `Strategy` as a concept allows us to re-think the way we approach Ember Engines as well. We can start thinking about engines as a strategy of certain type. In fact, it's very similar to `ConcatenationStrategy` described above.

+ reduces the API surface (as opposed to `ember-engines` and `ember-asset-loader`, only one strategy)
+ makes things explicit, meaning that you have to register a strategy

## Registering custom strategies

Ember applications would be able to register custom strategies via `ember-cli-build.js` file by passing a `strategies` property to the constructor.

Passing `strategies` as an option to the constructor will clobber default strategies that Ember CLI provides out of the box (they mimic existing behaviour of creating `dist` folder with final assets).

However, Ember CLI will expose an array of default strategies that people can take advantage of. Note, that `defaultStrategies` will be freezed so you can't push directly onto it. One would need to create a new array.

For example:

```javascript
// ember-cli-build.js
const { defaultStrategies } = require('ember-cli/lib/strategies');

module.exports = function(defaults) {
  const firstStrategy = new MyCustomStrategy();
  const secondStrategy = new MyCustomStrategy();
  const stragies = defaultStrategies.concat(firstStrategy, secondStrategy);

  const app = new EmberApp(defaults, { strategies });

  return app.toTree();
}
```

Note, that `strategies` is an optional property. If you don't pass it to the constructor or `strategies` is an empty array, you're effectively "opting out" of the feature and using the default behaviour.

This change should be behind an experiment flag, `STRATEGIES`. This will allow us to start experimenting with different strategies right away and not being tied to a particular release cycle.

#  Topics for Future RFCs

While working on this RFC, some ideas were brought into focus regarding existing and new features in Ember CLI. They all likely require separate discussions in future RFCs, but the discussion points have been included below.

## Treeshaking

Firstly, what's _tree-shaking_? AFAIK, the term [originated](https://groups.google.com/forum/#!msg/comp.lang.lisp/6zpZsWFFW18/-z_8hHRAIf4J) in Lisp. The gist of the idea is "how about we start using _only_ the code that we actually need?"

Secondly, how is it different from [Dead Code Elimination](https://en.wikipedia.org/wiki/Dead_code_elimination)? [Rich Harris](https://twitter.com/Rich_Harris) [offers](https://medium.com/@Rich_Harris/tree-shaking-versus-dead-code-elimination-d3765df85c80) a pretty good explanation in the context of [Rollup](https://rollupjs.org/). The gist is dead code elimination happens on a final product by removing bits that are unused. Treeshaking is not about excluding the code but about including the code that's used.

With this RFC, we lay out the foundation and create a framework by which both dead code elimination and treeshaking code be implemented.

However, there are still several things that are missing:

+ **Linker** - Responsible for resolving and reducing the graph to a tree containing only reachable modules.
+ **File System Resolver** - Responsible for connecting a module name with a file path.

`Linker` would be responsible for:

+ building a minimal dependency graph as well as check for redundant edges in the graph (more on the topic, [Transitive reduction of a directed graph](https://en.wikipedia.org/wiki/Transitive_reduction#Graph_algorithms_for_transitive_reduction));
+ producing an application tree with only used modules

Dependency graph represents dependencies using module names, there is a need to be able to convert module name to file path. This is where `File System Resolver` comes in. Here's couple of examples:

```javascript
fileSystemResolver.resolve('lodash') => `some-path/node_modules/lodash/lodash.js`
fileSystemResolver.resolve('ember-ajax') => `some-path/addon-modules/ember-ajax/index.js`
fileSystemResolver.resolve('ember-data') => `some-path/addon-modules/modules/ember-data/index.js`
fileSystemResolver.resolve('ember-data/-private') => `some-path/addon-modules/modules/-private.js`
```

This effort could be broken down into several phases:

+ dead modules elimination inside of the `addons/` (application would be the main entry point and unused modules are removed only from `addons/`)
+ dead modules elimination inside of the `app/`
  + removing unused components and helpers (requires analysing templates)
  + removing unused initializers/services (this likely entails work on dependency injection layer as we would need access to a resolver resolution map)
+ treeshaking (Rollup-like treeshaking where we include *only* the code that is used)

`Linker` would be able to take an `exclude` list of modules as a parameter. Although, valuable in some situations, it should be clearly marked as advanced API. It should be used as a last resort and serve as an "escape hatch".

It would make sense to implement `Linker` as a strategy. Developers would be able to "opt in"/"opt out" of optimising behaviour.

## Deprecating `app.import` API

Ember applications which choose to use `Linker` strategy should be able to remove usages of `app.import`.

## Tools

With growing complexity of Ember applications, it is crucial to provide more insights into final assets.

Main goals are:

- report raw/uglified/compressed asset sizes; [broccoli-concat-analyser](https://github.com/stefanpenner/broccoli-concat-analyser)
- find source code duplication across your javascript assets (enables you to
  fine tune code splitting parameters to reduce bundle invalidation rates as well
  as improve repeat page load performance)

# How We Teach This

This is a backward compatible change to the existing Ember CLI eco system. In order to teach users how to use strategies, we need to update the API docs with a section for this and the best practices of when to use this. A more general purpose blog post could be beneficial as well.

# Drawbacks

There are several potential drawbacks that are worth noting.

_Build performance_. There is minimal overhead in instantiating strategies and calling methods on them and I believe this approach shouldn't degrade build performance.

_Coupling with Broccoli_. The idea of having a more generic cli was discussed during last Ember CLI core face to face meeting and it was agreed that it was a sizeable amount of work. That kind of refactoring CLI would require a lot of code changes and is a large body of work. At the moment, Broccoli is a de facto Ember CLI build tool. We don't introduce any extra coupling, not more than it is right now.

_A note on add-ons_. Add-ons don't rely on the way Ember CLI does bundling. That means existing build system continues to work as expected and add-ons won't have to change their implementation.

_A note on non-javascript bundles_. Strategies could be used to create non-javascript bundles as well. For example, we could create a separate bundle for Handlebars templates if need be. I'm going to leave technical details out as they are out of the scope of this RFC.

# Alternatives

This RFC allows us to introduce _any_ bundling strategy when needed. [Webpack](https://webpack.js.org) has become very popular in solving this similar problem. We could implement a `WebpackStrategy` that would take care of the bundling. Ultimately, we need something that is aware of how Ember apps are assembled and how Ember apps utilize dependency injection that takes advantage of existing tools. The long term plan is to have a dependency graph that is aware of application structure so can avoid the "wall of configuration" that other asset packaging systems are susceptible to.

# Unresolved questions

+ Will there be an ordering problem (strategies)?
+ Will it increase build time?
+ Which strategies are going to be supported out-of-the-box?
+ Should we introduce the same API on add-on level?
+ Will simple extension strategy “just work”?
+ Communication between strategies
+ Debugging strategies and output validation

# Thanks

Many thanks for [@stefanpenner](https://github.com/stefanpenner/), [@rwjblue](https://github.com/rwjblue/) and [@chadhietala](https://github.com/chadhietala/) for helping me to drive this forward.