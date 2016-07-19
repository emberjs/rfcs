- Start Date: 2016-07-19
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Reduce complexity in Ember and its dependencies' build pipelines by
standardizing on the JavaScript module syntax.

# Motivation

Though many users think of Ember as a single entity, under the hood it
is composed of many different small packages.  Due to fragmentation in
the JavaScript ecosystem, these packages use different build pipelines,
module formats, output formats, and naming conventions.  This
fragmentation creates significant overhead for contributors, as they
must familiarize themselves with the quirks of each package's build
system and module format. Typically, this information is severely
underdocumented if it's documented at all.

Additionally, this complexity leaks out to anyone who wishes to consume
these packages. Rather than adhering to current conventions and best
practices for distributing code on npm, users must learn the vagaries of
how to consume each package; whether that's by consuming a global,
consuming anonymous AMD, named AMD, or something else. Ember's build
system accrues needless complexity because it has to handle all of these
different formats distributed in different ways.

Lastly, due to efforts like FastBoot, users expect to be able to run
their JavaScript code both in the browser and in Node.js.  We need a
system that makes our packages portable, so that they can be tested and
run in any environment.

# Detailed Design

This RFC proposes that all packages under the Ember.js umbrella be
incrementally migrated to standard JavaScript (ES6) modules.  This
format is trivially transpilable into whatever ultimate module system
the end user desires, whether that's {named, anonymous} AMD, CommonJS,
or
no module system at all using something like Rollup's bundle builds.

JavaScript modules have the benefit of being statically analyzable, so
tools can determine the dependency graph at build time.  By normalizing
Ember's dependencies into JavaScript modules, we make it much easier to
eliminate code that is not actually needed in final builds.

To facilitate a backwards-compatible migration, we will standardize on a
directory layout for projects, as well as a directory layout for the
final built output. All packages will be distributed via npm.

To ensure standardized behavior across repositories, a reusable npm
package `broccoli-module-alchemist` will be created to encapsulate the
decisions of this RFC.

The source code for broccoli-module-alchemist is available at
<https://github.com/monegraph/broccoli-module-alchemist>.

## Supported Authoring Formats

Authors may write package source code in TypeScript, ES2015, or
a mix of both. Because transpilation is done using the TypeScript
compiler, which supports both, projects can incrementally migrate
towards TypeScript, or use it only in portions where it makes sense.

Whether using TypeScript or JavaScript, authors should always use
standard ES2015 module syntax for both importing and exporting modules.

## Target Formats

The original source code is compiled from TypeScript or ES2015+ to ES5
to ensure maximum compatibility across browser and Node.js versions.

Out of the box, several different module formats are emitted. Output
targets can be configured on a package-by-package basis. For example, a
package that only targets Node.js may want to author in TypeScript and
only emit CommonJS-compatible modules.

### CommonJS

For use by Node.js and tools like browserify/webpack.

### JavaScript (ES2015/ES6) Modules

The standard JavaScript module syntax specified in ES2015, this
statically-analyzable format is intended to be used by tools like Rollup
or Ember CLI, which can analyze an entire app and its dependencies and
strip out unused code.

For Ember's packages, this is the output target that Ember will consume.

Note that only the import/export syntax remains in the output; 
all other ES2015 syntax is transpiled into ES5. This makes it easy for
tools like Rollup to consume without having to add a transpiler like
Babel.

### UMD

UMD modules can be consumed as CommonJS, AMD, or by exporting a global.

#### Using Default Export as Namespace (e.g. `$()` and `$.getJSON()`)

Some libraries designed before standard JavaScript modules use a pattern
where the default export is a function or other value, and additional
functionality is added via methods on that value.

For example, in jQuery, you can create a jQuery object using
`$('.some-selector')`; in other words, the global `$` is a function you
invoke to access jQuery functionality. But there are additional methods
on that function, such as `$.getJSON('/users.json')`, to access
secondary functionality.

Abusing JavaScript's default/named exports to re-create this is
error-prone due to circular dependencies, so transpilers don't support
this.

For packages that need to support a legacy API of this kind, they may
create a specially-named `src/index-global.js` file. Only for the UMD
build, this file will be used as the package's "entry point"; you can
manually add named exports to the default export and re-export it.

For example, jQuery's `src/index-global.js` might look like this:

```js
import jQuery, { getJSON } from "./index";

jQuery.getJSON = getJSON;

export default jQuery;
```

This would cause the build package to export the `jQuery` function as
the global `jQuery`, with the `getJSON` method added on to it.

## File System Layout

Packages will adopt the following layout:

* `src` - Source ES2015+ or TypeScript. Layout inside `src` is up to
  authors.
* `dist` - Per Broccoli defaults, built output is placed in the `dist`
  directory. The `package.json`, tests, and other configuration
  information should consume the files in `dist`.
  * `js` - JavaScript standard/ES2015/ES6 module syntax build
  * `cjs` - CommonJS build
  * `umd/package-name.js` - UMD build

## Build Pipeline

Packages will standardize on using Ember CLI as their build tool. This
maximizes familiarity for Ember developers becoming contributors to the
framework.

We will provide a package on npm, `broccoli-module-alchemist`, that
encapsulates these concerns. Using it in a package is as simple as:

1. `npm install ember-cli --save-dev`
2. `npm install broccoli-module-alchemist --save-dev`
3. Require `broccoli-module-alchemist` from `ember-cli-build.js` and
   return it as the Broccoli tree.
4. Run `ember build`

Here's an example build file:

```js
// ember-cli-build.js

var alchemist = require('broccoli-module-alchemist');

module.exports = function() {
  return alchemist();
}
```

### Under the Hood

We use two tools to produce the desired output formats:

1. The TypeScript compiler, which converts TypeScript and ES2015+ to
   ES5, in both CommonJS and standard module syntax.
2. Rollup, to package the ES5 source into legacy bundle formats like
   UMD.

# How We Teach This

Because this is an internal infrastructure concern, it does not have
user-facing impact. That said, we should ensure that ease-of-use and
maintainability are priorities to enable new contributors to get
started.

Because users who need this information are intermediate to advanced,
this RFC itself should serve as a good starting point. Additionally,
configuration information and how-tos will be incorporated into
`broccoli-module-alchemist`'s README.

# Drawbacks

The biggest drawback is that this change may introduce temporary
instability into the ecosystem. Many packages may rely on non-standard
semantics or syntax, and updating them may divert work that could go
elsewhere.

Additionally, this RFC does not go so far as attempting to standardize
the testing infrastructure. Some packages use mocha, while others use
QUnit. Some run their tests in Node.js, others in PhantomJS. Ideally, a
future RFC could attempt this unification; although rewriting entire
test suites just to use a different test harness does not seem like the
best use of precious time and energy.

# Alternatives

The obvious alternative is to do nothing; the current system works,
although it does preclude several nice optimizations. In terms of
alternate module systems, there does not seem to be a better subset than
JavaScript modules. JavaScript modules are the only standard to be
statically analyzable; it is much more difficult to translate from a
runtime-dependent format (like CommonJS) into a static format (like JS
modules) than vice versa.

# Unresolved questions

- What is the exact file system layout?
- How do we handle "multi-package" packages, like Glimmer, which include
  multiple logical packages in a `packages` directory? Should each
  package have its own `package.json`?
- Does it make sense to TypeScript-enable the infrastructure, given that
  many new ecosystem packages rely on it?
- Is it possible to conventionalize testing? If the only two systems in
  common use are Mocha and QUnit, can we support both?

# Related Reading

- [The Struggles of Publishing a JavaScript
  Library](https://nolanlawson.com/2015/10/19/the-struggles-of-publishing-a-javascript-library/)
