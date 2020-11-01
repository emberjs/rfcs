---
Start Date: 2016-11-05
RFC PR: https://github.com/emberjs/rfcs/pull/176
Ember Issue: (leave this empty)

---

# Summary

Make Ember feel less overwhelming to new users, and make Ember applications
start faster, by replacing the `Ember` global with a first-class system for
importing just the parts of the framework you need.

# Motivation

ECMAScript 2015 (also known as ES2015 or ES6) introduced a syntax for importing
and exporting values from modules. Ember aggressively adopted modules, and if
you've used Ember before, you're probably familiar with this syntax:

```js
import Ember from "ember";
import Analytics from "../mixins/analytics";

export default Ember.Component.extend(Analytics, {
  // ...
});
```

One thing to notice is that the entire Ember framework is imported as a single
package. Rather than importing `Component` directly, for example, you import
`Ember` and subclass `Ember.Component`. (And this example still works even if
you forget the import, because we also create a global variable called
`Ember`.)

Using Ember via a monolithic package or global object is not ideal for several
reasons:

* It's overwhelming for learners. There's a giant list of classes and functions
  with no hints about how they're related. The API documentation reflects this.
* Experienced developers who don't want all of Ember's features feel like
  they're adding unnecessary and inescapable bloat to their application.
* The `Ember` object must be built at boot time, requiring that we ship the
  entire framework to the browser. This has two major costs:
  1. Increased download time, particularly noticeable on slower connections.
  2. Increased parsing/evaluation cost, which still must be paid even when
     assets are cached. On some browsers/devices, this can far exceed the cost of the
     download itself.

Defining a public API for importing parts of Ember via JavaScript modules helps
us lay the groundwork for solving all of these problems.

#### Reducing Load Time

Modules help us eliminate unneeded code. The module syntax is _statically
analyzable_, meaning that a tool like Ember CLI can analyze an application's
source code and reliably determine which files, in both the framework and the
application, are actually needed. Anything that's not needed is omitted from the
final build.

This allows us to provide the file size benefits of a "small modules" approach
to building web applications while retaining the productivity benefits of a
complete, opinionated framework.

For example, if your application never used the `Ember.computed.union` computed
property helper, Ember could detect this and remove its code when you build your
application. This technique for slimming down the payload automatically is often
referred to as _tree shaking_ or _dead code elimination_.

Building the module graph doesn't just mean we get a list of files used by the application—
we also know which files are used _route-by-route_.

We can use this knowledge to optimize boot time even more, by prioritizing
sending only the JavaScript needed for the requested route, rather than the
entire application.

For example, if the user requests the URL
`https://app.example.com/articles/123`, the server could first send the code for
`ArticlesRoute`, the `Article` model, the `articles` template, and any
components and framework code used in the route. Only after the route is
rendered would we start to send the remainder of the application and framework
code in the background.

#### Guiding Learners

We can group framework classes and utilities by functionality, making it clear
what things are related and how they should work together. People can feel
confident they are getting only what they need at that moment, not an entire
framework that they're not sure they're benefiting from.

#### Modernizing Ember

Lastly, developers are growing increasingly accustomed to using JavaScript
modules to import libaries. If we don't adapt to modules, Ember will feel clunky
and antiquated compared to modern alternatives.

### Prior Art

Initial efforts to define a module API for Ember began with the
[`ember-cli-shims`][shims] addon. This addon provides a set of "shim" modules
that re-export a value off the global `Ember`. While this setup doesn't offer
the benefits of true modules, it did allow us to rapidly experiment with a
module API without making changes to Ember core.

[shims]: https://github.com/ember-cli/ember-cli-shims

Common feedback from shim users was that, while they were a net improvement,
they introduced too much verbosity and were hard for beginners to remember.

An oft-cited example of this verbosity is that implementing an object and using
`Ember.get` and `Ember.set` requires three different imports:

```js
import EmberObject from "ember-object";
import get from "ember-metal/get";
import set from "ember-metal/set";
```

In fact, one of the principles outlined in this RFC is designed to correct this
verbosity; namely, that [utility functions and the class they are related to
should share a module](#utility-functions-are-named-exports).

For those who have already adopted modules via the `ember-cli-shims` package, we
will provide a migration tool to rewrite shim modules into the final module API.
The static nature of the import syntax makes this even easier and more reliable
than migrating globals-based apps. The upgrade process should take no more than
a few minutes (see [Migration](#migration)).

This RFC also builds significantly on [@zeppelin's](https://github.com/zeppelin)
previous [ES6 modules RFC](https://github.com/emberjs/rfcs/pull/68), which drove
initial discussion, including the idea to use scoped packages.

# Detailed Design

## Terminology

* **Package** - a bundle of JavaScript addressable by npm and other package
  managers, it may contain many modules (but has a default module, usually
  called `index.js`).
* **Scoped Package** - a namespaced package whose name starts with an `@`, like
  `import Thing from "@scope/thing"`.
* **Module** - a JavaScript file with at least one default export or named
  export.
* **Top-Level Module** - the module provided by importing a package directly,
  like `import Component from "@ember/component"`.
* **Nested Module** - a module provided at a path _inside_ a package, like
  `import { addObserver } from "@ember/object/observers"`.

## Module Naming & Organization

Because our goal is to eliminate the `Ember` global object, any public classes,
functions or properties that currently exist on the global need an equivalent
module that can be imported.

Given how fundamental modules are to the development process, how we organize
and name them impacts new learners and seasoned veterans alike. Thus we must try
to find a balance between predictability for new and intermediate users, and
terseness for experienced developers with large apps.

There is another goal at play: we would like to help dispel the misconception
that Ember is a monolithic framework. Ideally, module names help us tell a story
about Ember's layered features. Rather than inheriting the entire framework at
once, you can pull in just the pieces you need.

For that reason, package names should assist the developer in understanding what
capabilities are added by bringing in that new package. We should pick
meaningful names, not let our public API be a by-product of how Ember's
internals are organized.

A full table of proposed mappings from global to module is available in
[Addendum 1 - Table of Module Names and Exports by
Global](#addendum-1---table-of-module-names-and-exports-by-global) and [Addendum
2 - Table of Module Names and Exports by
Package](#addendum-2---table-of-module-names-and-exports-by-package). Because
there is some implicit functionality that you get when loading Ember that is not
encapsulated in a global property (for example, automatically adding prototype
extensions), there is also [Addendum 3 - Table of Modules with Side
Effects](#addendum-3---table-of-modules-with-side-effects).

Before diving in to these tables, however, it may be helpful to understand some
of the thinking that guided this proposal. And keep in mind, this RFC specifies
a _baseline_ module API. Nothing here precludes adding additional models in the
future, as we discover missing pieces.

### Use Scoped Packages

Last year, [npm introduced support for scoped packages][scoped-packages]. Scopes
are similar to organizations on GitHub. They allow us to use any package name,
even if it's already in use on npm, by namespacing it inside a scope.

[scoped-packages]: http://blog.npmjs.org/post/116936804365/solving-npms-hard-problem-naming-packages

For example, the [`component`][component] package is already reserved by an
unmaintained tool; we couldn't use `component` as a package name even if we
wanted to.

[component]: https://www.npmjs.com/package/component

However, scopes allow us to create a package named `component` that lives under
the `@ember` scope: `import Component from "@ember/component"`.

The advantages of using scoped packages, as this proposal does, are two-fold:

1. "Official" packages are clearly differentiated from community packages.
2. There is no risk of naming conflicts with existing community packages.

Note that actually publishing packages to npm may not be immediately necessary
to implement this RFC. We should still design around this constraint so that we
have the option available to us in the future. For more discussion, see the
[Distribution unresolved question](#distribution).

### Prefer Common Terminology

Module names should use terms people are more likely to be familiar with. For
example, instead of the ambiguous `platform`, polyfills should be in a module
called `polyfill`.

Similarly, the vast majority of advanced Ember developers couldn't crisply
articulate the difference between `ember-metal` and `ember-runtime`. Instead, we
should prefer `ember-object`, to match how people actually talk about these
features: the Ember object model.

### Organize by Mental Model

One of the biggest barriers to learning is the fact that short-term memory is
limited. To understand a complex system like a modern web application, the
learner must hold in their head many different concepts—more concepts than most
people can reason about at once.

[Chunking][chunking] is a strategy for dealing with this. It means that you
present concepts that are conceptually related together. When the learner needs
to reason about the overall system, in their mind they can replace a group of
related concepts with a single, overarching concept.

[chunking]: https://en.wikipedia.org/wiki/Chunking_(psychology)

For example, if you tell someone that in order to build an Ember app, they will
need to understand computed properties, actions (bubbling and closure),
components, containers, registries, routes, helpers (stateful and stateless),
dependent keys, controllers, route maps, observers, transitions, mixins,
computed property macros, injected properties, the run loop, and array
proxies—they will rightfully feel like Ember is an overwhelming, overcomplicated
framework. Most people (your RFC author included) simply cannot keep this many
discrete concepts in their head at once.

The day-to-day reality of building an Ember app, of course, is not nearly so
complex. For those developers who stick through the learning curve, they end up
with a greatly simplified mental model.

This proposal attempts to re-align module naming with that simplified mental
model, placing everything into packages based on the chunk of functionality they
provide:

* `@ember/application` - Application-level concerns, like bootstrapping,
  initializers, and dependency injection.
* `@ember/component` - Classes and utilities related to UI components.
* `@ember/routing` - Classes used for multi-page routing.
* `@ember/service` - Classes and utilities for cross-cutting services.
* `@ember/controller` - Classes and utilities related to controllers.
* `@ember/object` - Classes and utilities related to Ember's object model,
  including `Ember.Object`, computed properties and observers.
* `@ember/runloop` - Methods for scheduling behavior on to the run loop.

It includes a few other packages that, over time, your author hopes become
either unneeded or can be moved outside of core into standalone packages:

* `@ember/array` - Array utilities and observation. Ideally these can be replaced
  with a combination of ES2015+ features and array diffing in Glimmer.
* `@ember/enumerable` - Replaced by iterables in ES2015.
* `@ember/string` - String formatting utilities (dasherize, camelize, etc.).
* `@ember/map` - Replaced by `Map` and `WeakMap` in ES2015.
* `@ember/polyfills` - Polyfills for `Object.keys`, `Object.assign` and `Object.create`.
* `@ember/utils` - Grab bag of utilities that could likely be replaced with
  something like lodash.

And finally, some packages that may be used by internals, extensions, or addons
but are not used day-to-day by app developers:

* `@ember/instrumentation` - Instrumentation hooks for measuring performance.
* `@ember/debug` - Utility functions for debugging, and hooks used by debugger tools like Ember Inspector.

### Classes are Default Exports

Classes that the user is supposed to import and subclass are always the default
export, never a named export. In the case where a package has more than one primary class,
those classes live in a nested module.

This rule ensures there is no ambiguity about whether something is a named
export or a default export: classes are always default exports. In tandem with
the following rule ([Utility Functions are Named
Exports](#utility-functions-are-named-exports)), this also means that classes
and the functions that act on them are grouped into the same `import` line.

#### Examples

Primary class only:

```js
import EmberObject from "@ember/object";
```

Primary class plus secondary classes:

```js
import Component from "@ember/component";
import Checkbox from "@ember/component/checkbox";

import Map from "@ember/map";
import MapWithDefault from "@ember/map/with-default";
```

Multiple primary classes:

```js
import Router from "@ember/routing/router";
import Route from "@ember/routing/route";
```

### Utility Functions are Named Exports

Functions that are only useful with a particular class, or are used most frequently with
that class, are named exports from the package that exports the class.

#### Examples

```js
import Service, { inject } from "@ember/service";
import EmberObject, { get, set } from "@ember/object";
```

In cases where there are many utility functions associated with a class, they can be further subdivided into
nested packages but remain named exports:

```js
import EmberObject, { get, set } from "@ember/object";
import { addObserver } from "@ember/object/observers";
import { addListener } from "@ember/object/events";
```

In the future, [decorators][decorators] would be included under this rule as
well. In fact, designing with an eye towards decorators was a large driver
behind this principle. For more discussion, see the [Everything is a Named
Export alternative](#everything-is-a-named-export).

[decorators]: http://tc39.github.io/proposal-decorators/

### One Level Deep

To avoid deep directory hierarchies with mostly-empty directories, this proposal
limits nesting inside a top-level package to a single level. Deep nesting like
this can add additional time to navigating the hierarchy without adding much
benefit.

Java packages often have this problem due to their URL-based namespacing; see
e.g. [this Java
library](https://github.com/elvishew/xLog/tree/fbfb60f9472e32723436b3d6bdd6c1878a5afb37/library/src)
where you end up with deeply nested directories, like
`xLog/library/src/test/java/com/elvishew/xlog/printer/AndroidPrinterTest.java`.

This rule leads to including the type in the name of the module in some cases
where it might otherwise be grouped instead. For example, instead of
`@ember/routing/locations/none`, we prefer `@ember/routing/none-location` to
avoid the second level of nesting.

### No Non-Module Namespaces

The global version of Ember includes several functions that also act as a
namespace to group related functionality.

For example, `Ember.run` can be used to run some code inside a run loop, while
`Ember.run.scheduleOnce` is used to schedule a function onto the run loop once.

Similarly, `Ember.computed` can be used to indicate a method should be treated as
a computed property, but computed property macros also live on `Ember.computed`, like
`Ember.computed.alias`.

When consumed via modules, these functions no longer act as a namespace. That's
because tacking these secondary functions on to the main function requires us to
eagerly evaluate them (not to mention the potential deoptimizations in
JavaScript VMs by adding properties to a function object).

In practice, that means that this won't work:

```js
// Won't work!
import { run } from "@ember/runloop";
run.scheduleOnce(function() {
  // ...
});
```

Instead, you'd have to do this:

```js
import { scheduleOnce } from "@ember/runloop";
scheduleOnce(function() {
  // ...
});
```

The [migration tool](#migration), described below, is designed to detect these
cases and migrate them correctly.

### Prototype Extensions and Other Code with Side Effects

Some parts of Ember change global objects rather than exporting classes or
functions. For example, Ember (by default) installs additional methods on
`String.prototype`, like the `camelize()` method.

Any code that has side effects lives in a module without any exports; importing
the module is enough to produce the desired side effects. For example, if I
wanted to make the string extensions available to the application, I could
write:

```js
import "@ember/extensions/string"
```

Generally speaking, modules that have side effects are harder to debug and can
cause compatibility issues, and should be avoided if possible.

## Migration

To assist in assessing this RFC in real-world applications, and to help upgrade
apps should this RFC be accepted and implemented, your author has provided an
automatic migration utility, or "codemod":

[ember-modules-codemod][ember-modules-codemod]

To run the codemod, `cd` into an existing Ember app and run the following commands.

```sh
npm install ember-modules-codemod -g
ember-modules-codemod
```

**Note**: The codemod currently requires Node 6 or later to run.

This codemod uses [`jscodeshift`](https://github.com/facebook/jscodeshift) to
update an Ember application in-place to the module syntax proposed in this RFC.
It can update apps that use the global `Ember`, and will eventually also support
apps using [ember-cli-shims][shims].

[shims]: https://github.com/ember-cli/ember-cli-shims

**Make sure you save any changes in your app before running the codemod, because
it modifies files in place. Obviously, because this RFC is speculative, your app
will not function after applying this codemod. For now, the codemod is only
useful for assessing how this proposal looks in real-world applications.**

For example, it will rewrite code that looks like this:

```js
import Ember from 'ember';

export default Ember.Component.extend({
  isAnimal: Ember.computed.or('isDog', 'isCat')
});
```

Into this:

```js
import Component from '@ember/component';
import { or } from '@ember/object/computed';

export default Component.extend({
  isAnimal: or('isDog', 'isCat')
});
```

For more information, see the [README][ember-modules-codemod].

[ember-modules-codemod]: https://github.com/tomdale/ember-modules-codemod

# How We Teach This

This RFC makes changes to one of the most foundational (and historically stable)
concepts in Ember: how you access framework code. Because of that, it is hard to
overstate the impact these changes will have. We need to proceed carefully to
avoid confusion and churn.

It is possible that the work required to update the documentation and other
learning materials will be significantly more than the work required to do the
actual implementation. That means we need to start getting ready _now_, so that
when the code changes are ready, it is not blocked by a big documentation
effort.

That said, we do have the advantage of the new modules being "just JavaScript."
We can lean heavily on the greater JavaScript community's learning materials,
and any teaching we do has the benefit of being transferable and not an
"Ember-only" skill.

## Documentation Examples

Examples in the Getting Started tutorial, guides and API docs will need to be
updated to the new module syntax.

Probably the most efficient and least painful way to do this would be to write a
tool that can extract code snippets from Markdown files and run the
[migrator](#migration) on them, then replace the extracted code with the updated
version. For the API docs, this tool would need to be able to handle Markdown embedded
in JSDoc-style documentation.

The benefit of this approach is that, once we have verified the script works
reliably, we can wait until the last possible moment to make the switch. If we
attempt to update everything by hand, the duration and tediousness of that
process will likely take out an effective "lock" on the documentation code base,
where people will put off making big changes because of the potential for merge
conflicts.

## Generators

Generators are used by new users to help them get a handle on the framework, and
by experienced users to avoid typing repetitive boilerplate. We need to ensure
that the generators that ship with Ember are updated to use modules as soon as
they are ready. The recent work by the Ember CLI team to ship generators with
the Ember package itself, rather than Ember CLI, should make this relatively
painless.

## API Documentation

Our API documentation has long been a source of frustration, because the laundry
list of (often rarely used or internal) classes makes Ember feel far more
overwhelming than it really is.

The shift to modules gives us a good opportunity to rethink the presentation of
our API documentation. Instead of the imposing mono-list, we should group the
API documentation by package–which, conveniently in this proposal, means they
will also be grouped by area of functionality.

We should investigate the broader ecosystem to see if there is a good tool that
generates package-oriented documentation for JavaScript projects. If not, we may
wish to adapt an existing tool to do so.

## Explaining the Migration

Once the guides and API documentation are updated, modules should be
straightforward for new learners—indeed, more and more new learners are starting
with JavaScript modules as the baseline.

The most challenging aspect of teaching the new modules API, counterintuitively,
will likely be _existing_ users. In particular, for changes that touch nearly
every file, most teams working on large apps cannot pause work for a week to
implement the change.

Our focus needs to be:

* Communicating clearly that the existing global build will work for the
  foreseeable future.
* Making clear the file size benefits of moving to modules.
* Building robust tooling that allows even large apps to migrate in a day or
  two, not a week.

It is important to frame the module transition as a carrot, not a stick. We
should avoid dire warnings or deprecation notices. Instead, we should provide
good reporting when doing Ember CLI builds. If an app is compiled in globals
mode, we can offer suggestions for how to reduce the file size, providing a
helpful pointer to the modules migration guide. This will make the transition
feel less like churn and more like an optimization opportunity that developers
can take advantage of when they have the time or resources.

### Addons

One pitfall is that a _single_ use of the `Ember` global means we have to
include the entire framework. That means that a developer could migrate their
entire app to modules, but a single old addon that uses the Ember globals will
negate the benefits.

This requires a two-pronged strategy:

* Tight integration into Ember CLI
  * Good reporting to make it obvious when a fallback to globals mode
  occurs, and which addons/files are causing it.
  * An opt-in mode to prohibit globals mode. Installing an incompatible addon
    would produce an error.
* Community outreach and pull requests to help authors update addons.

# Drawbacks

## Complexity

There is something elegantly simple about a single `Ember` global that contains
everything. Introducing multiple packages means you don't just have to know what
you need—you also need to know where to import it from.

JavaScript module syntax is also something not everyone will be familiar with,
given its newness. However, this is something we must deal with anyway because
module syntax is already in use within apps.

## Module Churn

The `ember-cli-shims` package is already included by default in new Ember apps,
and is in fairly common usage. Many developers are already familiar with its
API. This drawback can be at least partially mitigated by [the automated
migration process](#migration), which will be easily applied to existing shimmed
apps.

## Scoped Packages Are an Unknown Quantity

This proposal relies on scoped packages. Despite being released over a year ago,
scoped packages are not always well supported.

For example, [scoped packages currently wreak havoc on Yarn][yarn]. Until very
recently, the [npmjs.com](https://npmjs.com) search did not include scoped
packages. Generally speaking, there will be a long-tail of tools in the
ecosystem that will choke on scoped packages.

That said, Angular 2 is distributed under the `@angular` scope, and TypeScript
recently adopted the `@types` scope for publishing TypeScript typings to npm.
The popularity of both of these should drive compatibility. Despite this, we can
expect [similar compatibility issues][scoped-proxy-issue] for some time.

[yarn]: https://github.com/yarnpkg/yarn/issues?utf8=✓&q=is%3Aissue%20is%3Aopen%20scoped%20packages
[scoped-proxy-issue]: https://github.com/angular/angular/issues/8422

## Nested Modules

To satisfy the [Classes are Default Exports](#classes-are-default-exports) rule,
this RFC proposes the use of nested modules. That is, a module name may contain
an additional path segment beyond the package name. For example,
`@ember/object/observers` is a nested module, while `@ember/object` is not.

In the Node/CommonJS world, nested modules are unusual but not unheard of. For
example, Lodash offers a [functional programming
style](https://github.com/lodash/lodash/wiki/FP-Guide) accessed by calling
`require('lodash/fp')`.

There are two drawbacks associated with nested modules:

1. Because they are uncommon, developers may be confused by the syntax.
2. Because they allow you to "reach in" to the package for an
   arbitrary file, encouraging the end user to use nested modules may
   inadvertently _also_ encourage them to access private modules, thinking they are
   public.

The first issue is surmountable with education, good reference documentation,
and good tools to help guide developers in the right direction. That this style
is uncommon in the Node ecosystem seems to be more a [function of
dogma](http://blog.izs.me/post/44149270867/why-no-directorieslib-in-node-the-less-snarky)
than any technical shortcoming of nested modules.

To ensure that developers don't inadvertently access private modules, we have
two good options:

1. Package modules in such a way that private modules _cannot_ be accessed.
2. Take a page from Ember Data and put all private modules in a `-private`
   directory, hopefully making it clear accessing this module is not playing by
   the rules.

We could avoid using this uncommon style by hoisting nested modules up to their
own package. For example, `@ember/object/observers` could become
`@ember/observers` or `@ember/object-observers`. However, because I could not
find a strong technical reason against it, and because having more packages is
in tension with the explicit goal to [make Ember feel less
overwhelming](#organize-by-mental-model), I decided it was worth the small cost.

# Alternatives

### `ember-` prefix

One alternative to the `@ember` scope is to use the `ember-` prefix. This avoids
the drawbacks around scoped packages described above. However, they would be
indistinguishable from the large number of community packages that begin with
`ember-`.

### Everything is a Named Export

This proposal argues that classes should be a module's default export, and any
utility functions should be a named export. That means you can never have more
than one class per module, and _that_ means, inherently, more `import`
statements than a system where multiple classes can live in one module.

Additionally, in cases where there is not a clear "primary" class, this can feel
a little awkward:

```js
import Route from "@ember/routing/route";
import Router from "@ember/routing/router";
```

One commonly proposed alternative is to say that classes become named exports,
and default exports are not used at all. The above example would become:

```js
import { Route, Router } from "@ember/routing";
```

In this case, classes are distinguished by being capitalized, rather than by
being a default export.

There is one major change coming to JavaScript and Ember that, your author
believes, deals a fatal blow to this approach: decorators.

If you're unfamiliar with decorators, see [Addy Osmani's great
overview][decorators]. Decorators provide a mechanism for adding declarative
annotations to classes, methods, properties and functions.

[decorators]: https://medium.com/google-developers/exploring-es7-decorators-76ecb65fb841#.y9umddvsl

For example, Robert Jackson has an [experimental library for using decorators to
annotate computed properties in a class][ember-computed-decorators]. Something
like this will probably make its way into Ember in the future:

```js
import EmberObject, { computed } from "@ember/object";

export default class Cat extends EmberObject {
  @computed("hairLength")
  isDomesticShortHair(hairLength) {
    return hairLength < 3;
  }
}
```

[ember-computed-decorators]: https://github.com/rwjblue/ember-computed-decorators

Most decorators are tightly coupled to a particular class because they configure
some aspect of behavior that is only relevant to that class. If every decorator
and every class share a namespace, it is hard to identify which go with each
other.

```js
import { Router, Route, resource, model, location, inject, queryParam } from "@ember/routing";
```

Can you tell me which of these decorators goes with which class?

And this import is getting so long, you'd probably be tempted to break it up
into multiple lines _anyway_, so it's not clear that it's actually a win over
separate imports.

Contrast this with the same thing expressed using the rules in
this RFC:

```js
import Router, { resource, location } from "@ember/routing/router";
import Route, { model, inject, queryParam } from "@ember/routing/route";
```

Here, the decorators are clearly tied to their class. And it's far nicer from a
refactoring perspective: if you delete a class from a file, you then delete a
single line from your imports.

Contrast that with making fiddly edits to a long list of named exports unrelated
to each other.

# Unresolved Questions

### Intimate APIs

How much do we want to provide module API for so-called "intimate
APIs"—technically private, but in widespread use?

### Backwards Compatibility for Addons

How do we provide an API for addons to use modules but fall back to globals mode
in older versions of Ember? We should ensure that, at minimum, addons can
continue to support LTS releases. At the same time, it's critical that adding an
addon doesn't opt your entire application back in to the entire framework.

Because there is a lot of implementation-specific detail to get right here, and
because it doesn't otherwise block landing this module naming RFC, the final
design of API for addon authors should be broken out into a separate RFC.

### Distribution

In practice, how do we ship this code to end users of Ember CLI?

When building client-side apps, it's very important to avoid duplicate
dependencies, which can quickly cause file size to balloon out of control.

Unfortunately, npm@3's de-duping is so naïve that it's likely that users would
end up in dependency hell if we shipped the framework as separate npm packages.
There's no good way to ship dependencies in version lockstep and feel confident
that they will reliably be de-duped.

Until Yarn usage is more widespread, and to eliminate significant complexity in
the first iteration, it probably makes sense for the first phase of
implementation to continue shipping a single npm package that Ember CLI apps can
depend on. This gives us atomic updates and makes sure you never have one piece
of the framework interacting with a different piece that is inadvertently three
versions old.

What this means is that, rather than shipping `@ember/object` on npm, we'd ship
a single `ember-source` (or something) package that includes the entire
framework. At build time, the Ember build process would virtually map the
`@ember/object` package to the right file inside `ember-source`. In essence, all
of the benefits of smaller bundles without the boiling hellbroth of managing
dependencies.

That said, because this RFC is designed with an eye towards eventually
publishing each package to npm individually, we will have that option available
to us in the future once we determine that we can do so without causing lots of
pain.

# Addenda

_(Ed. note: These tables are automatically generated from the scripts in the [codemod][codemod] repository.)_

[codemod]: https://github.com/tomdale/ember-modules-codemod

## Addendum 1 - Table of Module Names and Exports by Global

| Before                                | After                                                                      |
| ---                                   | ---                                                                        |
| `Ember.$`                             | `import $ from "jquery"`                                                   |
| `Ember.A`                             | `import { A } from "@ember/array"`                                         |
| `Ember.Application`                   | `import Application from "@ember/application"`                             |
| `Ember.Array`                         | `import EmberArray from "@ember/array"`                                    |
| `Ember.ArrayProxy`                    | `import ArrayProxy from "@ember/array/proxy"`                              |
| `Ember.AutoLocation`                  | `import AutoLocation from "@ember/routing/auto-location"`                  |
| `Ember.Checkbox`                      | `import Checkbox from "@ember/component/checkbox"`                         |
| `Ember.Component`                     | `import Component from "@ember/component"`                                 |
| `Ember.ContainerDebugAdapter`         | `import ContainerDebugAdapter from "@ember/debug/container-debug-adapter"` |
| `Ember.Controller`                    | `import Controller from "@ember/controller"`                               |
| `Ember.DataAdapter`                   | `import DataAdapter from "@ember/debug/data-adapter"`                      |
| `Ember.DefaultResolver`               | `import GlobalsResolver from "@ember/application/globals-resolver"`        |
| `Ember.Enumerable`                    | `import Enumerable from "@ember/enumerable"`                               |
| `Ember.Evented`                       | `import Evented from "@ember/object/evented"`                              |
| `Ember.HashLocation`                  | `import HashLocation from "@ember/routing/hash-location"`                  |
| `Ember.Helper`                        | `import Helper from "@ember/component/helper"`                             |
| `Ember.Helper.helper`                 | `import { helper } from "@ember/component/helper"`                         |
| `Ember.HistoryLocation`               | `import HistoryLocation from "@ember/routing/history-location"`            |
| `Ember.LinkComponent`                 | `import LinkComponent from "@ember/routing/link-component"`                |
| `Ember.Location`                      | `import Location from "@ember/routing/location"`                           |
| `Ember.Map`                           | `import EmberMap from "@ember/map"`                                        |
| `Ember.MapWithDefault`                | `import MapWithDefault from "@ember/map/with-default"`                     |
| `Ember.Mixin`                         | `import Mixin from "@ember/object/mixin"`                                  |
| `Ember.MutableArray`                  | `import MutableArray from "@ember/array/mutable"`                          |
| `Ember.NoneLocation`                  | `import NoneLocation from "@ember/routing/none-location"`                  |
| `Ember.Object`                        | `import EmberObject from "@ember/object"`                                  |
| `Ember.RSVP`                          | `import RSVP from "rsvp"`                                                  |
| `Ember.Resolver`                      | `import Resolver from "@ember/application/resolver"`                       |
| `Ember.Route`                         | `import Route from "@ember/routing/route"`                                 |
| `Ember.Router`                        | `import Router from "@ember/routing/router"`                               |
| `Ember.Service`                       | `import Service from "@ember/service"`                                     |
| `Ember.String.camelize`               | `import { camelize } from "@ember/string"`                                 |
| `Ember.String.capitalize`             | `import { capitalize } from "@ember/string"`                               |
| `Ember.String.classify`               | `import { classify } from "@ember/string"`                                 |
| `Ember.String.dasherize`              | `import { dasherize } from "@ember/string"`                                |
| `Ember.String.decamelize`             | `import { decamelize } from "@ember/string"`                               |
| `Ember.String.fmt`                    | `import { fmt } from "@ember/string"`                                      |
| `Ember.String.htmlSafe`               | `import { htmlSafe } from "@ember/string"`                                 |
| `Ember.String.loc`                    | `import { loc } from "@ember/string"`                                      |
| `Ember.String.underscore`             | `import { underscore } from "@ember/string"`                               |
| `Ember.String.w`                      | `import { w } from "@ember/string"`                                        |
| `Ember.TextArea`                      | `import TextArea from "@ember/component/text-area"`                        |
| `Ember.TextField`                     | `import TextField from "@ember/component/text-field"`                      |
| `Ember.addListener`                   | `import { addListener } from "@ember/object/events"`                       |
| `Ember.addObserver`                   | `import { addObserver } from "@ember/object/observers"`                    |
| `Ember.aliasMethod`                   | `import { aliasMethod } from "@ember/object"`                              |
| `Ember.assert`                        | `import { assert } from "@ember/debug"`                                    |
| `Ember.assign`                        | `import { assign } from "@ember/polyfills"`                                |
| `Ember.cacheFor`                      | `import { cacheFor } from "@ember/object/internals"`                       |
| `Ember.compare`                       | `import { compare } from "@ember/utils"`                                   |
| `Ember.computed`                      | `import { computed } from "@ember/object"`                                 |
| `Ember.computed.alias`                | `import { alias } from "@ember/object/computed"`                           |
| `Ember.computed.and`                  | `import { and } from "@ember/object/computed"`                             |
| `Ember.computed.bool`                 | `import { bool } from "@ember/object/computed"`                            |
| `Ember.computed.collect`              | `import { collect } from "@ember/object/computed"`                         |
| `Ember.computed.deprecatingAlias`     | `import { deprecatingAlias } from "@ember/object/computed"`                |
| `Ember.computed.empty`                | `import { empty } from "@ember/object/computed"`                           |
| `Ember.computed.equal`                | `import { equal } from "@ember/object/computed"`                           |
| `Ember.computed.filter`               | `import { filter } from "@ember/object/computed"`                          |
| `Ember.computed.filterBy`             | `import { filterBy } from "@ember/object/computed"`                        |
| `Ember.computed.filterProperty`       | `import { filterProperty } from "@ember/object/computed"`                  |
| `Ember.computed.gt`                   | `import { gt } from "@ember/object/computed"`                              |
| `Ember.computed.gte`                  | `import { gte } from "@ember/object/computed"`                             |
| `Ember.computed.intersect`            | `import { intersect } from "@ember/object/computed"`                       |
| `Ember.computed.lt`                   | `import { lt } from "@ember/object/computed"`                              |
| `Ember.computed.lte`                  | `import { lte } from "@ember/object/computed"`                             |
| `Ember.computed.map`                  | `import { map } from "@ember/object/computed"`                             |
| `Ember.computed.mapBy`                | `import { mapBy } from "@ember/object/computed"`                           |
| `Ember.computed.mapProperty`          | `import { mapProperty } from "@ember/object/computed"`                     |
| `Ember.computed.match`                | `import { match } from "@ember/object/computed"`                           |
| `Ember.computed.max`                  | `import { max } from "@ember/object/computed"`                             |
| `Ember.computed.min`                  | `import { min } from "@ember/object/computed"`                             |
| `Ember.computed.none`                 | `import { none } from "@ember/object/computed"`                            |
| `Ember.computed.not`                  | `import { not } from "@ember/object/computed"`                             |
| `Ember.computed.notEmpty`             | `import { notEmpty } from "@ember/object/computed"`                        |
| `Ember.computed.oneWay`               | `import { oneWay } from "@ember/object/computed"`                          |
| `Ember.computed.or`                   | `import { or } from "@ember/object/computed"`                              |
| `Ember.computed.readOnly`             | `import { readOnly } from "@ember/object/computed"`                        |
| `Ember.computed.reads`                | `import { reads } from "@ember/object/computed"`                           |
| `Ember.computed.setDiff`              | `import { setDiff } from "@ember/object/computed"`                         |
| `Ember.computed.sort`                 | `import { sort } from "@ember/object/computed"`                            |
| `Ember.computed.sum`                  | `import { sum } from "@ember/object/computed"`                             |
| `Ember.computed.union`                | `import { union } from "@ember/object/computed"`                           |
| `Ember.computed.uniq`                 | `import { uniq } from "@ember/object/computed"`                            |
| `Ember.copy`                          | `import { copy } from "@ember/object/internals"`                           |
| `Ember.create`                        | `import { create } from "@ember/polyfills"`                                |
| `Ember.debug`                         | `import { debug } from "@ember/debug"`                                     |
| `Ember.deprecate`                     | `import { deprecate } from "@ember/application/deprecations"`              |
| `Ember.deprecateFunc`                 | `import { deprecateFunc } from "@ember/application/deprecations"`          |
| `Ember.get`                           | `import { get } from "@ember/object"`                                      |
| `Ember.getOwner`                      | `import { getOwner } from "@ember/application"`                            |
| `Ember.getProperties`                 | `import { getProperties } from "@ember/object"`                            |
| `Ember.guidFor`                       | `import { guidFor } from "@ember/object/internals"`                        |
| `Ember.inject.controller`             | `import { inject } from "@ember/controller"`                               |
| `Ember.inject.service`                | `import { inject } from "@ember/service"`                                  |
| `Ember.inspect`                       | `import { inspect } from "@ember/debug"`                                   |
| `Ember.instrument`                    | `import { instrument } from "@ember/instrumentation"`                      |
| `Ember.isArray`                       | `import { isArray } from "@ember/array"`                                   |
| `Ember.isBlank`                       | `import { isBlank } from "@ember/utils"`                                   |
| `Ember.isEmpty`                       | `import { isEmpty } from "@ember/utils"`                                   |
| `Ember.isEqual`                       | `import { isEqual } from "@ember/utils"`                                   |
| `Ember.isNone`                        | `import { isNone } from "@ember/utils"`                                    |
| `Ember.isPresent`                     | `import { isPresent } from "@ember/utils"`                                 |
| `Ember.keys`                          | `import { keys } from "@ember/polyfills"`                                  |
| `Ember.makeArray`                     | `import { makeArray } from "@ember/array"`                                 |
| `Ember.observer`                      | `import { observer } from "@ember/object"`                                 |
| `Ember.on`                            | `import { on } from "@ember/object/evented"`                               |
| `Ember.onLoad`                        | `import { onLoad } from "@ember/application"`                              |
| `Ember.platform.defineProperty`       | `import { defineProperty } from "@ember/polyfills"`                        |
| `Ember.platform.hasPropertyAccessors` | `import { hasPropertyAccessors } from "@ember/polyfills"`                  |
| `Ember.removeListener`                | `import { removeListener } from "@ember/object/events"`                    |
| `Ember.removeObserver`                | `import { removeObserver } from "@ember/object/observers"`                 |
| `Ember.reset`                         | `import { reset } from "@ember/instrumentation"`                           |
| `Ember.run`                           | `import { run } from "@ember/runloop"`                                     |
| `Ember.run.begin`                     | `import { begin } from "@ember/runloop"`                                   |
| `Ember.run.bind`                      | `import { bind } from "@ember/runloop"`                                    |
| `Ember.run.cancel`                    | `import { cancel } from "@ember/runloop"`                                  |
| `Ember.run.debounce`                  | `import { debounce } from "@ember/runloop"`                                |
| `Ember.run.end`                       | `import { end } from "@ember/runloop"`                                     |
| `Ember.run.join`                      | `import { join } from "@ember/runloop"`                                    |
| `Ember.run.later`                     | `import { later } from "@ember/runloop"`                                   |
| `Ember.run.next`                      | `import { next } from "@ember/runloop"`                                    |
| `Ember.run.once`                      | `import { once } from "@ember/runloop"`                                    |
| `Ember.run.schedule`                  | `import { schedule } from "@ember/runloop"`                                |
| `Ember.run.scheduleOnce`              | `import { scheduleOnce } from "@ember/runloop"`                            |
| `Ember.run.throttle`                  | `import { throttle } from "@ember/runloop"`                                |
| `Ember.runInDebug`                    | `import { runInDebug } from "@ember/debug"`                                |
| `Ember.runLoadHooks`                  | `import { runLoadHooks } from "@ember/application"`                        |
| `Ember.sendEvent`                     | `import { sendEvent } from "@ember/object/events"`                         |
| `Ember.set`                           | `import { set } from "@ember/object"`                                      |
| `Ember.setOwner`                      | `import { setOwner } from "@ember/application"`                            |
| `Ember.setProperties`                 | `import { setProperties } from "@ember/object"`                            |
| `Ember.subscribe`                     | `import { subscribe } from "@ember/instrumentation"`                       |
| `Ember.tryInvoke`                     | `import { tryInvoke } from "@ember/utils"`                                 |
| `Ember.trySet`                        | `import { trySet } from "@ember/object"`                                   |
| `Ember.typeOf`                        | `import { typeOf } from "@ember/utils"`                                    |
| `Ember.unsubscribe`                   | `import { unsubscribe } from "@ember/instrumentation"`                     |
| `Ember.warn`                          | `import { warn } from "@ember/debug"`                                      |

## Addendum 2 - Table of Module Names and Exports by Package

Each package is sorted by module name, then export name.

### `@ember/application`
| Module                                                              | Global                  |
| ---                                                                 | ---                     |
| `import Application from "@ember/application"`                      | `Ember.Application`     |
| `import { getOwner } from "@ember/application"`                     | `Ember.getOwner`        |
| `import { onLoad } from "@ember/application"`                       | `Ember.onLoad`          |
| `import { runLoadHooks } from "@ember/application"`                 | `Ember.runLoadHooks`    |
| `import { setOwner } from "@ember/application"`                     | `Ember.setOwner`        |
| `import { deprecate } from "@ember/application/deprecations"`       | `Ember.deprecate`       |
| `import { deprecateFunc } from "@ember/application/deprecations"`   | `Ember.deprecateFunc`   |
| `import GlobalsResolver from "@ember/application/globals-resolver"` | `Ember.DefaultResolver` |
| `import Resolver from "@ember/application/resolver"`                | `Ember.Resolver`        |

### `@ember/array`
| Module                                            | Global               |
| ---                                               | ---                  |
| `import EmberArray from "@ember/array"`           | `Ember.Array`        |
| `import { A } from "@ember/array"`                | `Ember.A`            |
| `import { isArray } from "@ember/array"`          | `Ember.isArray`      |
| `import { makeArray } from "@ember/array"`        | `Ember.makeArray`    |
| `import MutableArray from "@ember/array/mutable"` | `Ember.MutableArray` |
| `import ArrayProxy from "@ember/array/proxy"`     | `Ember.ArrayProxy`   |

### `@ember/component`
| Module                                                | Global                |
| ---                                                   | ---                   |
| `import Component from "@ember/component"`            | `Ember.Component`     |
| `import Checkbox from "@ember/component/checkbox"`    | `Ember.Checkbox`      |
| `import Helper from "@ember/component/helper"`        | `Ember.Helper`        |
| `import { helper } from "@ember/component/helper"`    | `Ember.Helper.helper` |
| `import TextArea from "@ember/component/text-area"`   | `Ember.TextArea`      |
| `import TextField from "@ember/component/text-field"` | `Ember.TextField`     |

### `@ember/controller`
| Module                                       | Global                    |
| ---                                          | ---                       |
| `import Controller from "@ember/controller"` | `Ember.Controller`        |
| `import { inject } from "@ember/controller"` | `Ember.inject.controller` |

### `@ember/debug`
| Module                                                                     | Global                        |
| ---                                                                        | ---                           |
| `import { assert } from "@ember/debug"`                                    | `Ember.assert`                |
| `import { debug } from "@ember/debug"`                                     | `Ember.debug`                 |
| `import { inspect } from "@ember/debug"`                                   | `Ember.inspect`               |
| `import { runInDebug } from "@ember/debug"`                                | `Ember.runInDebug`            |
| `import { warn } from "@ember/debug"`                                      | `Ember.warn`                  |
| `import ContainerDebugAdapter from "@ember/debug/container-debug-adapter"` | `Ember.ContainerDebugAdapter` |
| `import DataAdapter from "@ember/debug/data-adapter"`                      | `Ember.DataAdapter`           |

### `@ember/enumerable`
| Module                                       | Global             |
| ---                                          | ---                |
| `import Enumerable from "@ember/enumerable"` | `Ember.Enumerable` |

### `@ember/instrumentation`
| Module                                                 | Global              |
| ---                                                    | ---                 |
| `import { instrument } from "@ember/instrumentation"`  | `Ember.instrument`  |
| `import { reset } from "@ember/instrumentation"`       | `Ember.reset`       |
| `import { subscribe } from "@ember/instrumentation"`   | `Ember.subscribe`   |
| `import { unsubscribe } from "@ember/instrumentation"` | `Ember.unsubscribe` |

### `@ember/map`
| Module                                                 | Global                 |
| ---                                                    | ---                    |
| `import EmberMap from "@ember/map"`                    | `Ember.Map`            |
| `import MapWithDefault from "@ember/map/with-default"` | `Ember.MapWithDefault` |

### `@ember/object`
| Module                                                      | Global                            |
| ---                                                         | ---                               |
| `import EmberObject from "@ember/object"`                   | `Ember.Object`                    |
| `import { aliasMethod } from "@ember/object"`               | `Ember.aliasMethod`               |
| `import { computed } from "@ember/object"`                  | `Ember.computed`                  |
| `import { get } from "@ember/object"`                       | `Ember.get`                       |
| `import { getProperties } from "@ember/object"`             | `Ember.getProperties`             |
| `import { observer } from "@ember/object"`                  | `Ember.observer`                  |
| `import { set } from "@ember/object"`                       | `Ember.set`                       |
| `import { setProperties } from "@ember/object"`             | `Ember.setProperties`             |
| `import { trySet } from "@ember/object"`                    | `Ember.trySet`                    |
| `import { alias } from "@ember/object/computed"`            | `Ember.computed.alias`            |
| `import { and } from "@ember/object/computed"`              | `Ember.computed.and`              |
| `import { bool } from "@ember/object/computed"`             | `Ember.computed.bool`             |
| `import { collect } from "@ember/object/computed"`          | `Ember.computed.collect`          |
| `import { deprecatingAlias } from "@ember/object/computed"` | `Ember.computed.deprecatingAlias` |
| `import { empty } from "@ember/object/computed"`            | `Ember.computed.empty`            |
| `import { equal } from "@ember/object/computed"`            | `Ember.computed.equal`            |
| `import { filter } from "@ember/object/computed"`           | `Ember.computed.filter`           |
| `import { filterBy } from "@ember/object/computed"`         | `Ember.computed.filterBy`         |
| `import { filterProperty } from "@ember/object/computed"`   | `Ember.computed.filterProperty`   |
| `import { gt } from "@ember/object/computed"`               | `Ember.computed.gt`               |
| `import { gte } from "@ember/object/computed"`              | `Ember.computed.gte`              |
| `import { intersect } from "@ember/object/computed"`        | `Ember.computed.intersect`        |
| `import { lt } from "@ember/object/computed"`               | `Ember.computed.lt`               |
| `import { lte } from "@ember/object/computed"`              | `Ember.computed.lte`              |
| `import { map } from "@ember/object/computed"`              | `Ember.computed.map`              |
| `import { mapBy } from "@ember/object/computed"`            | `Ember.computed.mapBy`            |
| `import { mapProperty } from "@ember/object/computed"`      | `Ember.computed.mapProperty`      |
| `import { match } from "@ember/object/computed"`            | `Ember.computed.match`            |
| `import { max } from "@ember/object/computed"`              | `Ember.computed.max`              |
| `import { min } from "@ember/object/computed"`              | `Ember.computed.min`              |
| `import { none } from "@ember/object/computed"`             | `Ember.computed.none`             |
| `import { not } from "@ember/object/computed"`              | `Ember.computed.not`              |
| `import { notEmpty } from "@ember/object/computed"`         | `Ember.computed.notEmpty`         |
| `import { oneWay } from "@ember/object/computed"`           | `Ember.computed.oneWay`           |
| `import { or } from "@ember/object/computed"`               | `Ember.computed.or`               |
| `import { readOnly } from "@ember/object/computed"`         | `Ember.computed.readOnly`         |
| `import { reads } from "@ember/object/computed"`            | `Ember.computed.reads`            |
| `import { setDiff } from "@ember/object/computed"`          | `Ember.computed.setDiff`          |
| `import { sort } from "@ember/object/computed"`             | `Ember.computed.sort`             |
| `import { sum } from "@ember/object/computed"`              | `Ember.computed.sum`              |
| `import { union } from "@ember/object/computed"`            | `Ember.computed.union`            |
| `import { uniq } from "@ember/object/computed"`             | `Ember.computed.uniq`             |
| `import Evented from "@ember/object/evented"`               | `Ember.Evented`                   |
| `import { on } from "@ember/object/evented"`                | `Ember.on`                        |
| `import { addListener } from "@ember/object/events"`        | `Ember.addListener`               |
| `import { removeListener } from "@ember/object/events"`     | `Ember.removeListener`            |
| `import { sendEvent } from "@ember/object/events"`          | `Ember.sendEvent`                 |
| `import { cacheFor } from "@ember/object/internals"`        | `Ember.cacheFor`                  |
| `import { copy } from "@ember/object/internals"`            | `Ember.copy`                      |
| `import { guidFor } from "@ember/object/internals"`         | `Ember.guidFor`                   |
| `import Mixin from "@ember/object/mixin"`                   | `Ember.Mixin`                     |
| `import { addObserver } from "@ember/object/observers"`     | `Ember.addObserver`               |
| `import { removeObserver } from "@ember/object/observers"`  | `Ember.removeObserver`            |

### `@ember/polyfills`
| Module                                                    | Global                                |
| ---                                                       | ---                                   |
| `import { assign } from "@ember/polyfills"`               | `Ember.assign`                        |
| `import { create } from "@ember/polyfills"`               | `Ember.create`                        |
| `import { defineProperty } from "@ember/polyfills"`       | `Ember.platform.defineProperty`       |
| `import { hasPropertyAccessors } from "@ember/polyfills"` | `Ember.platform.hasPropertyAccessors` |
| `import { keys } from "@ember/polyfills"`                 | `Ember.keys`                          |

### `@ember/routing`
| Module                                                          | Global                  |
| ---                                                             | ---                     |
| `import AutoLocation from "@ember/routing/auto-location"`       | `Ember.AutoLocation`    |
| `import HashLocation from "@ember/routing/hash-location"`       | `Ember.HashLocation`    |
| `import HistoryLocation from "@ember/routing/history-location"` | `Ember.HistoryLocation` |
| `import LinkComponent from "@ember/routing/link-component"`     | `Ember.LinkComponent`   |
| `import Location from "@ember/routing/location"`                | `Ember.Location`        |
| `import NoneLocation from "@ember/routing/none-location"`       | `Ember.NoneLocation`    |
| `import Route from "@ember/routing/route"`                      | `Ember.Route`           |
| `import Router from "@ember/routing/router"`                    | `Ember.Router`          |

### `@ember/runloop`
| Module                                          | Global                   |
| ---                                             | ---                      |
| `import { begin } from "@ember/runloop"`        | `Ember.run.begin`        |
| `import { bind } from "@ember/runloop"`         | `Ember.run.bind`         |
| `import { cancel } from "@ember/runloop"`       | `Ember.run.cancel`       |
| `import { debounce } from "@ember/runloop"`     | `Ember.run.debounce`     |
| `import { end } from "@ember/runloop"`          | `Ember.run.end`          |
| `import { join } from "@ember/runloop"`         | `Ember.run.join`         |
| `import { later } from "@ember/runloop"`        | `Ember.run.later`        |
| `import { next } from "@ember/runloop"`         | `Ember.run.next`         |
| `import { once } from "@ember/runloop"`         | `Ember.run.once`         |
| `import { run } from "@ember/runloop"`          | `Ember.run`              |
| `import { schedule } from "@ember/runloop"`     | `Ember.run.schedule`     |
| `import { scheduleOnce } from "@ember/runloop"` | `Ember.run.scheduleOnce` |
| `import { throttle } from "@ember/runloop"`     | `Ember.run.throttle`     |

### `@ember/service`
| Module                                    | Global                 |
| ---                                       | ---                    |
| `import Service from "@ember/service"`    | `Ember.Service`        |
| `import { inject } from "@ember/service"` | `Ember.inject.service` |

### `@ember/string`
| Module                                       | Global                    |
| ---                                          | ---                       |
| `import { camelize } from "@ember/string"`   | `Ember.String.camelize`   |
| `import { capitalize } from "@ember/string"` | `Ember.String.capitalize` |
| `import { classify } from "@ember/string"`   | `Ember.String.classify`   |
| `import { dasherize } from "@ember/string"`  | `Ember.String.dasherize`  |
| `import { decamelize } from "@ember/string"` | `Ember.String.decamelize` |
| `import { fmt } from "@ember/string"`        | `Ember.String.fmt`        |
| `import { htmlSafe } from "@ember/string"`   | `Ember.String.htmlSafe`   |
| `import { loc } from "@ember/string"`        | `Ember.String.loc`        |
| `import { underscore } from "@ember/string"` | `Ember.String.underscore` |
| `import { w } from "@ember/string"`          | `Ember.String.w`          |

### `@ember/utils`
| Module                                     | Global            |
| ---                                        | ---               |
| `import { compare } from "@ember/utils"`   | `Ember.compare`   |
| `import { isBlank } from "@ember/utils"`   | `Ember.isBlank`   |
| `import { isEmpty } from "@ember/utils"`   | `Ember.isEmpty`   |
| `import { isNone } from "@ember/utils"`    | `Ember.isNone`    |
| `import { isPresent } from "@ember/utils"` | `Ember.isPresent` |
| `import { tryInvoke } from "@ember/utils"` | `Ember.tryInvoke` |
| `import { typeOf } from "@ember/utils"`    | `Ember.typeOf`    |

### `jquery`
| Module                   | Global    |
| ---                      | ---       |
| `import $ from "jquery"` | `Ember.$` |

### `rsvp`
| Module                    | Global       |
| ---                       | ---          |
| `import RSVP from "rsvp"` | `Ember.RSVP` |

## Addendum 3 - Table of Modules with Side Effects

| Module                               | Description                                |
| ---                                  | ---                                        |
| `import "@ember/extensions"`         | Adds all of Ember's prototype extensions.  |
| `import "@ember/extensions/string"`  | Adds just `String` prototype extensions.   |
| `import "@ember/extensions/array"`   | Adds just `Array` prototype extensions.    |
| `import "@ember/extensions/function"`| Adds just `Function` prototype extensions. |
