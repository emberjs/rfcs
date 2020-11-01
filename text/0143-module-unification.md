---
Start Date: 2016-05-09
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/143
Tracking: https://github.com/emberjs/rfc-tracking/issues/27 / https://github.com/emberjs/ember.js/issues/14882

---

> Note: This RFC replaces the closely related RFC for [Module
Normalization](https://github.com/emberjs/rfcs/pull/124). As discussed in the
[Alternatives](#alternatives) section below, many concepts are shared between
the two proposals, but there is also a fundamental difference.

# Summary

Create a unified pattern for organizing and naming modules in Ember projects
that is deterministic, extensible, and ergonomic.

# Motivation

Ember CLI's conventions for project layouts and file naming are central to every
Ember developer's experience. It's crucial to get both the technical and
ergonomic details right.

The existing conventions used by Ember CLI have evolved gradually and
organically over the years. Ember CLI and its predecessor Ember App Kit were
early adopters of ES modules and have always leveraged strong conventions to
deduce an understanding of modules based on their locations. Ember CLI's
resolver encodes those conventions to enable run-time module resolutions.

The current system works fairly well, but has some complexities and
inconsistencies that both steepen its learning curve and limit its technical
potential.

Drawbacks include:

* Confusion over which of two orthogonal approaches to use for organizing
  modules:

  * classic - modules are organized at the top-level by "type"
    (`components`, `templates`, etc.) and then by namespace and name.

  * pods - modules are organized by namespace, then name, then type.

* Addons define modules to be merged into an application through a special `app`
  directory. These public modules are typically private modules that are
  imported and re-exported, which introduces an extra module per export and an
  extra level of abstraction to learn.

* Because addons' modules are mixed into an application, there's the possibility
  of naming collisions between two addons or an addon and its consuming
  application.

* Modules don't have a clear sense of "locality", which prevents the ability to
  declare modules that are available only in a "local" namespace (this as-yet
  unsupported feature has been called "local lookup").

* Resolution rules that are declared only in JavaScript are difficult to
  analyze and optimize.

* Module resolution is inefficient due to the number of potential places to
  lookup a particular module by name.

Recognizing these drawbacks, the Core Team compiled a set of
[design constraints](https://github.com/emberjs/core-notes/blob/master/ember.js/2016-01/january-22.md#summary-of-pods-design-constraints)
for a rethink of Ember's approach to modules:

1. Reasonable branching factor. Users should see a reasonable number of items at any given level in their hierarchy. Flattening out too much results in an unreasonably large number of items.
2. No slashes in component names. The existing system allows this, but we don't want to support invocation of nested components in Glimmer Components.
3. Addons need to participate in the naming scheme, most likely with namespace prefix and double-colon separator.
4. Subtrees should be relocatable. If you move a directory to a new place in the tree, its internal structure should all still work.
5. There should be no cognitive overhead when adding a new thing. The right way should be obvious and not impose unnecessary decisions.
6. We need clean separation between the namespace of the user's own components, helpers, routes, etc and the framework's own type names ("component", "helper", etc) so that we can disambiguate them and add future ones.
7. Ideally we will have a place to put tests and styles alongside corresponding components.
8. Local relative lookup for components and helpers needs to work.
9. Avoid the "titlebar problem", in which many files are all named "component.js" and you can't tell them apart in your editor.
10. The resolver should be configured via declarative rules, not imperative
    JavaScript. In addition to enforcing consistency, this allows addons to
    augment the system with their own types in a predictable way.
11. Module structures must be statically analyzable at build time to enable
    efficiency optimizations.
12. Module classifications must be extensible and allow for customizations by
    apps, engines, and addons.

> Note: Constraints > 9 were added based on discussions subsequent to the
initial meeting.

This proposal attempts to address these constraints with a single consistent
approach to modules that will make Ember easier to use and learn _and_ improve
the efficiency of Ember's resolver at run-time.

# Detailed Design

This proposal introduces a new top-level directory, `src`, and establishes
conventions for organizing modules within it. Also proposed is a refactor of the
Ember resolver to enable efficient and flexible resolutions based upon the new
module conventions.

The `src` directory will be used to contain the core ES modules within an Ember
CLI project, whether that project contains an application, addon, or engine. To
maintain backward compatibility, the `src` directory will be allowed to
co-exist alongside existing `app` and/or `addon` directories, although these
directories should eventually be deprecated.

## Examples

Let's start by taking a look at some examples of Ember projects organized
according to the proposed conventions.

### Example Application

A simple blogging application could be structured as follows:
```
src
├── data
│   ├── models
│   │   ├── comment
│   │   │   ├── adapter.js
│   │   │   ├── model.js
│   │   │   └── serializer.js
│   │   ├── post
│   │   │   ├── adapter.js
│   │   │   ├── model.js
│   │   │   └── serializer.js
│   │   └── author.js
│   └── transforms
│       └── date.js
├── init
│   ├── initializers
│   │   └── i18n.js
│   └── instance-initializers
│       └── auth.js
├── services
│   └── auth.js
├── ui
│   ├── components
│   │   ├── date-picker
│   │   │   ├── component.js
│   │   │   └── template.hbs
│   │   └── list-paginator
│   │       ├── paginator-control
│   │       │   ├── component.js
│   │       │   └── template.hbs
│   │       ├── component.js
│   │       └── template.js
│   ├── partials
│   │   └── footer.hbs
│   ├── routes
│   │   ├── application
│   │   │   └── template.hbs
│   │   ├── index
│   │   │   ├── controller.js
│   │   │   ├── route.js
│   │   │   └── template.hbs
│   │   └── posts
│   │       ├── -components
│   │       │   ├── -utils
│   │       │   │   └── strings.js
│   │       │   ├── capitalize.js
│   │       │   └── titleize.js
│   │       ├── post
│   │       │   ├── -components
│   │       │   │   └── post-viewer
│   │       │   │       ├── component.js
│   │       │   │       └── template.hbs
│   │       │   ├── edit
│   │       │   │   ├── -components
│   │       │   │   │   ├── post-editor
│   │       │   │   │   │   ├── post-editor-button
│   │       │   │   │   │   │   ├── component.js
│   │       │   │   │   │   │   └── template.hbs
│   │       │   │   │   │   ├── calculate-post-title.js
│   │       │   │   │   │   ├── component.js
│   │       │   │   │   │   └── template.hbs
│   │       │   │   │   ├── route.js
│   │       │   │   │   └── template.hbs
│   │       │   │   ├── route.js
│   │       │   │   └── template.hbs
│   │       │   ├── route.js
│   │       │   └── template.hbs
│   │       ├── route.js
│   │       └── template.hbs
│   ├── styles
│   │   └── app.scss
│   └── index.html
├── utils
│   └── md5.js
├── main.js
└── router.js
```

### Example Engine

An engine could provide the same blogging functionality with almost entirely the
same module structure as the example blog application. Only the following
notable changes would be needed:

* An engine should declare its routes in `src/routes.js` instead of `src/router.js`
* An engine would require a `dummy` app within `tests`
* An engine should export an `Engine` instead of an `Application` from `src/main.js`

### Example Addon

Here's how the
[ember-power-select](https://github.com/cibernox/ember-power-select) addon could
be restructured:

```
src
├── styles
│   └── ember-power-select.scss
├── ui
│   └── components
│       ├── main
│       │   ├── before-options
│       │   │   ├── component.js
│       │   │   └── template.hbs
│       │   ├── options
│       │   │   ├── component.js
│       │   │   └── template.hbs
│       │   ├── trigger
│       │   │   ├── component.js
│       │   │   └── template.hbs
│       │   ├── component.js
│       │   └── template.hbs
│       ├── multiple
│       │   ├── trigger
│       │   │   ├── component.js
│       │   │   └── template.hbs
│       │   ├── component.js
│       │   └── template.hbs
│       └── is-selected.js
└── main.js
```

### Migration Tool

As a proof of concept for the module layout described in this RFC, Robert
Jackson has created a [migration
tool](https://github.com/rwjblue/ember-module-migrator) and used it to migrate
the following repos:

* [Ghost admin client](https://github.com/rwjblue/--ghost-modules-sample/tree/grouped-collections/src)
* [Travis client](https://github.com/rwjblue/--travis-modules-sample/tree/modules/src)
* [`ember new my-app`](https://github.com/rwjblue/--new-app-blueprint/tree/modules/src)

You can also use Robert's migration tool on your own projects to gain a feel for
how this RFC will affect your work.

## ES Modules

It's important to understand how ES module paths are mapped from the file
system so that you can import modules from elsewhere in your project and its
associated dependencies.

ES module paths will be formed from a project's package name followed by a
direct mapping of file paths from the project root. The file's final extension
(e.g. `js` or `hbs`) will be excluded because all ES modules will of course be
compiled into JavaScript from their original format.

For example, the file `src/ui/components/date-picker.js` in the
`my-calendar` app will be exported with the module path
`my-calendar/src/ui/components/date-picker`.

An application and its associated addons and engines will all be merged into the
same ES module space, as is done today. Any module can import from any other
module within this space, although cross-package imports should be done with
care.

## Module Naming and Organization

This section describes the conventions proposed for naming and organizing a
project's modules within `src`. These conventions will allow Ember CLI's
resolver to determine the purpose of each module at run-time. They will also
enable static analysis of modules to lint against errors and to prepare a
normalized map for efficient resolutions.

Every resolvable module must have both a `name` and a `type`. The `type`
typically corresponds to the base class of the module's default export (e.g.
`route`, `template`, etc.).

Modules can be grouped together with other modules of related types in
"collections". Collections are directories with type-aware resolution rules
which allow related modules to share a namespace. For example, the `models`
collection contains models, adapters, and serializers.

Collections that are related to each other can be further organized in "group"
directories. For example, the `ui` group contains the `components`, `partials`,
and `routes` collections.

Ember CLI will have a build step that normalizes modules to a common form and
builds a mapping between that form and the ES module path described above.
While building this normalized map, the build must error and provide useful
messages if any module naming errors are detected. Unregistered collections and
types should not be allowed. Also, the same normalized module path must not be
repeated through alternative naming forms.

### Module Type

The type of a module can be determined through the following file naming and
module export rules:

1. `src/${type}` - typed modules named `main` (explained further below), in
    which default exports match the type specified by the file name.
2. `src/${collection}/${namespace}/${name}/${type}` - expanded collection
    modules, in which default exports match the type specified by the file name.
3. `src/${collection}/${namespace}/${name}` - in which type can be inferred
    based on the module's exports. Default exports must match the default
    type for the collection. If there is no default export, named exports will
    be scanned for a matching type allowed in the collection.

Note that template precompilers will need to use default vs. named exports
appropriately in order to satisfy the expectations of Rules 2 and 3.

Here are a few example applications of the module type determination rules:

```
// Rule 1

src/router (with `export default Ember.Router.extend()`)
=> name: 'main',
   type: 'router'

// Rule 2

src/ui/routes/posts/post/route.js (with `export default Ember.Route.extend()`)
=> collection: 'ui/routes',
   namespace: 'posts',
   name: 'post',
   type: 'route'

src/ui/routes/posts/post/template (with `export default Ember.HTMLBars.template(COMPILED)`)
=> collection: 'ui/routes',
   namespace: 'posts',
   name: 'post',
   type: 'template'

// Rule 3

src/data/models/author (with `export default DS.Model.extend()`)
=> collection: 'data/models',
   name: 'author',
   type: 'model' (the default type for the models collection)

src/ui/components/titleize (with `export let helper = Ember.Helper.helper(function() { })`)
=> collection: 'ui/components',
   name: 'titleize',
   type: 'helper'

src/ui/components/show-title (with `export let template = Ember.HTMLBars.template(COMPILED)`)
=> collection: 'ui/components',
   name: 'show-title',
   type: 'template'
```

### Main Modules

Every project must have a "main" module, named `src/main.js`, that
serves as an entry point into the project.

The main module must export an `Application`, `Engine`, or (new) `Addon` class.
This class must define a `modulePrefix`, which must match the node package name
for the project.

The main module also declares other properties that help the Ember resolver
understand relationships between projects. For instance, the main module can
declare which modules in an addon are available to a consuming app's resolver.

The main module of an addon can also declare a `rootName`, which is used by the
resolver to lookup main modules. Initially, the `rootName` will be a read-only
property that equals the `modulePrefix` with any `ember-` and  `ember-cli-`
prefixes stripped (e.g. `ember-power-select` becomes `power-select`). It's
possible that we may allow overrides / aliases in the future.

Modules that appear alongside `main.js` in `src` are also considered `main`
modules for their respective `type`. For instance, `src/router.js` is registered
with a `name` of `main` and a `type` of `router`.

### Module Collections

Top-level namespaces within `src` serve to group modules into
type-aware "collections".

The following rules apply to module collections and types:

1. Each collection can contain one or more types. The types allowed
   in a particular collection MUST be explicitly declared.
2. Each type MAY exist in any number of collections.
3. Each type MUST have only one "definitive collection", which is the
   collection the resolver will use for resolutions if a module can't be found
   in the local (i.e. originating) collection.
4. Each collection MAY have a single "default type". If a module does not
   indicate its type through its file name, then its default export should
   align with the default type for its collection.
5. Each collection can allow "private collections" to be defined at a namespace.
   Private collections are localized additions to a top-level collection,
   available only from the namespace at which they're defined.
6. Top-level collections may be grouped for organization purposes. No
   resolvable modules must be placed in a group directory.
7. A collection can appear only once in a project (i.e. it can not be
   contained in multiple group directories, or in a group as well as at the
   top-level).

The following collections and allowed types (rules 1 & 2) are proposed:

* `components` - COMPONENT, HELPER, template
* `initializers` - INITIALIZER
* `instance-initializers` - INSTANCE-INITIALIZER
* `models` - MODEL, ADAPTER, SERIALIZER
* `partials` - PARTIAL
* `routes` - ROUTE, CONTROLLER, template
* `services` - SERVICE
* `transforms` - TRANSFORM
* `utils` - UTIL

> Note: ALL CAPS indicates which collections are definitive (rule 3) for a type.

The following default types are proposed for collections (rule 4):

* `components` - component
* `initializer` - initializer
* `instance-initializers` - instance-initializer
* `models` - model
* `partials` - partial
* `routes` - route
* `services` - service
* `transforms` - transform
* `utils` - util

The following private collections are allowed within collections (rule 5):

* `components` - utils
* `models` - utils
* `initializers` - utils
* `instance-initializers` - utils
* `routes` - components, utils
* `services` - utils
* `transforms` - utils

The following groups are proposed for collections (rule 6):

* `data` - models, transforms
* `init` - initializers, instance-initializers
* `ui` - components, partials, routes

The collection and type system is designed to be extensible, so that addons can
contribute their own collections and types. The `data` collection and its
corresponding types should be defined in ember-data. Liquid-fire might want to
define an `animations` collection and a `transition` type, and expand `routes`
to allow `animations` as a private collection.

The specific format of collection and type declarations for addons is TBD.

#### "Components" Collection

This proposal broadens the scope of the term "component" to include all
template-invocable parts of Ember. This includes today's components and helpers,
and the future implementation of "glimmer components" (with angle brackets) and
element modifiers.

Grouping template-invocable elements together in a single collection recognizes
that they already coexist in the same namespace. After all, only one helper OR
component can be invoked as `{{foo-bar}}`. Using a common collection will not
only simplify file management and searching, it will also provide implicit
linting against creating a helper and class-based component of the same name.

#### Private Collections

You may wish to make a component available in a particular template without
polluting the top-level `components` collection with a more local concern.
Private collections allow you to augment a top-level collection's contents for
use at a particular namespace.

Private collections are declared as a directory sharing the name of the
top-level collection, prefixed with a `-`. So the top-level `routes`
collection could be augmented via a private `-components` collection.

Say that you want to define a `post-viewer` component to be available only from
within `src/ui/routes/posts/post/template.hbs`. You could achieve this by
creating your component module in
`src/ui/routes/posts/post/-components/post-viewer.js`.

#### Non-resolved Files

The rules above apply to modules that are resolved, namely `*.js` and `*.hbs`
files. Other files that are used for documenting code, such as `*.md` and
`*.html` files, can be freely co-located in any directories.

Conventions will still be used for non-resolved files that have significance
within an Ember project, including:

* `src/ui/styles` - A project's stylesheets.
* `src/ui/index.html` - A project's html container.

### Packages

In-repo addons (including engines) will be placed in a new top-level `packages`
directory (a sibling of `src`). We can begin to use the term "packages" instead
of the rather clumsy "in-repo addons". This differentiation will emphasize that
packages are internal and addons are external to a project. Packages should be
seen as a lightweight way to add new namespacing within a project without the
overhead of a full addon.

The `packages` directory will provide a separate space away from other library
modules that might be kept in `lib`, the current directory used for in-repo
addons. Introducing a new top-level directory will allow a clear migration path
for in-repo addons, in the same way that there's a clear migration path from
`app` to `src`.

Inside `packages`, packages should be grouped by name. Each package can have
its own `index.js`, `package.json`, and `src` directory.

## Ember Resolver Refactor

The Ember resolver must be refactored significantly to be made aware of the
new `src` and `packages` directories and associated conventions.

### Module Normalization

As discussed above, Ember CLI will perform a normalization process for all the
modules in a project and its associated projects. The normalization step will
involve the construction of a map from each module's normalized form to its
corresponding ES module path. If any conflicts are detected, the process should
error and notify the developer.

The Ember resolver will only look up modules in their normalized form, utilizing
the pre-built normalization map to resolve the actual module path.

### Addon modules

A resolver will only implicitly consider an addon's top-level modules named
`main` (e.g. a `main` component) to be public and available for resolution. More
explicit control over an addon's public modules can be declared in the addon's
`main` module (details TBD). An addon's public modules will all be resolvable at
the `rootName` of the addon (see above).

Public components and helpers can be invoked in templates using the `rootName`
as a namespace. For modules named `main`, the bare root name will suffice.

Let's say that the `ember-power-select` addon has a `rootName` of `power-select`
and a top-level `main` component declared in `src/ui/components/main.js`. An
app could invoke this component in a template as `{{power-select::main}}` or
more simply as `{{power-select}}`.

Addons should use the same namespacing that will be used by consuming apps when
invoking their own components and helpers from templates. For instance, if the
`ember-power-select` addon has a `date-picker` component that invokes multiple
`main` components, it should also invoke them in a template as
`{{power-select::main}}` or more simply as `{{power-select}}`.

### Module Resolutions

Module resolution rules must account for the following:

* The requested module's `type`, `name`, and (potentially) `namespace`.
* (Optional) A "source" `rootName`, collection, and namespace from which the
  lookup originates.
* (Optional) An "associated type" for lookups that should start in a collection
  that is not definitive for the requested `type`.

Module resolutions occur in the following order:

1. Local - If a source module is specified and the requested type is allowed in
   the source module's collection, look in a namespace based on the source
   module's namespace + name.
2. Private - If a source module is specified, look in a private collection at
   the source module's namespace, if one exists that is definitive for the
   requested type.
3. Associated - If an associated type is specified, look in the definitive
   collection for that associated type. Only resolve if the collection can
   contain the requested type.
4. Top-level - In the definitive collection for the requested type, defined at
   its top-level.

The resolver must maintain mappings of modules at multiple levels to make these
resolutions efficient. A lookup tree can be pre-built for production builds.

#### Example Resolutions

Let's walk through some example resolutions from the above blogging app paired
with the `ember-power-select` addon. We'll assume that the package name for
the app is `blogmeister`, and the package name for the addon is
`ember-power-select`. The addon has a `rootName` of `power-select` for cleaner
references.

----

From `blogmeister/src/ui/components/list-paginator/template`:

`{{paginator-control}}` resolves to `blogmeister/src/ui/components/list-paginator/paginator-control/component`

`{{date-picker}}` resolves to `blogmeister/src/ui/components/date-picker/component`

`{{power-select}}` resolves to `ember-power-select/src/ui/components/main/component`

`{{power-select::multiple}}` resolves to `ember-power-select/src/ui/components/multiple/component`

----

From `blogmeister/src/routes/posts/post/template`:

`{{post-viewer}}` resolves to `blogmeister/src/ui/routes/posts/post/-components/post-viewer/component`

`{{date-picker}}` resolves to `blogmeister/src/ui/components/date-picker/component`

`{{power-select}}` resolves to `ember-power-select/src/ui/components/main/component`

## Other Refactorings

### Generators and Blueprints

Generators and blueprints will need to be made aware of the new module
conventions.

Let's take a look at the files that some generators will create (note: tests
have been left out of these examples for now):

`ember g component date-picker`:

* `src/ui/components/date-picker/component.js`
* `src/ui/components/date-picker/template.hbs`

`ember g component ui/routes/posts/post-editor`:

* `src/ui/routes/posts/-components/post-editor/component.js`
* `src/ui/routes/posts/-components/post-editor/template.hbs`

`ember g helper titleize`:

* `src/ui/components/titleize.js`

# How We Teach This

The Ember guides will need to be updated significantly to reflect the new
conventions.

## Teaching Conventions through Tooling

As discussed above, generators and blueprints will be made aware of the new
module conventions. This will help new projects start on track and stay on
track as modules are added.

Developers with existing projects will be able to use Robert Jackson's
[migration tool](https://github.com/rwjblue/ember-module-migrator) to move their
projects over to use the new conventions. This tool is a WIP and will continue
to be refined to work well with both the classic and pods structures. It's
possible these migration capabilities will eventually be rolled into Ember
Watson.

Furthermore, the Ember Inspector should be enhanced to understand the new
conventions and become more type and collection aware.

## New Concepts

It will be important for both new and experienced Ember developers to
understand some core concepts that are proposed in this RFC.

### Collections and Types

This proposal's concept of collections and types should feel familiar enough to
users of both the classic and pods layouts to enable a smooth transition. In
many ways, this proposal merges the classic and pods layouts into a single
uniform layout.

The core driver to collections is to store "like with like". However, instead of
the classic layout's narrow definition of "like" to be of a _single_ type, this
proposal takes the pods approach that _multiple_ types can be related. A good
test of whether multiple module types should be stored together is whether they
should be considered to share a common namespace. Routes, controllers, and
templates are a good example, as are models, adapters, and serializers.

A related concept to understand about collections is the notion of a default
type. Every top-level module within a collection can be considered to match its
default type (unless named exports are used in those modules to represent types
other than the default). Within a collection's namespaces, every module must be
either that default type or related to it. It's helpful to consider that every
namespace within a collection represents a set of named module exports, and that
the default type represents the default export for that collection.

Here's an illustration of exports from a collection:

```
src
  data
    models
      author.js <- exports an Author `model`, the default type in the `models` collection
      comment
        adapter.js     <- exports a Comment `adapter`
        model.js       <- exports a Comment `model`
        serializer.js  <- exports a Comment `serializer`
```

#### Components

The term "component" has been widely adopted across most front-end frameworks
to describe a broad swath of UI concerns. Using the same term for the collection
of template-invocable UI elements will lower the learning curve for developers
who are new to Ember, while allowing for a useful set of specialized terms to
flourish to describe particular _types_ of components.

We've already started down the road of component specialization by introducing
the concept of "routable components". Once we start actually using "routable
components" in practice, it will become necessary to refer to plain old
components as something more specific, like "template components". And this
distinction will probably lead to plain old helpers being referred to as
"template helpers". Other concepts, such as "Glimmer components" and "template
component modifiers" will soon be mixed in. We will end up with a multi-faceted
toolbox available at the template layer which deserves a simple name that
matches developer expectations. The general term "components" seems a good fit.

### Scope

Developers should understand the available levels of module scope, as well as
when each is appropriate to use. Scope should be considered when modules are
generated, and developers should feel free to move modules if they expand or
contract in scope.

The following levels of scope should be understood:

* Private - private collections should be used when a component or utility
  function is needed from a single namespace.

* Project - top-level, project-wide collections should be used for modules that
  are needed throughout a project.

* Local package - namespaced collections can be useful to group a common set of
  cross-cutting concerns within a project.

* Local engine - a type of local package that encapsulates a set of
  functionality that benefits from run-time isolation and strict dependency
  sharing.

### Testing

Unit, integration, and some acceptance tests can now be co-located with their
associated modules. Co-location should be encouraged because it makes test
modules easier to locate in the file system, and easier to move if a module's
scope changes.

Robert Jackson plans to adapt the
[Grand Testing Unification RFC](https://github.com/emberjs/rfcs/pull/119)
to illustrate test co-location and to introduce module types for tests.

# Drawbacks

Any change to a pattern as fundamental as file naming will incur some mental
friction for developers who are accustomed to the current conventions. It is
hoped that tooling like Robert's migrator and Ember Watson can lessen this
friction by automating transitions, and that updated guides, generators, and
blueprints can make these conventions easy to follow.

Of course, we won't prevent usage of the currently used patterns for some time,
but they will eventually be deprecated. Some efficiencies, especially in the
resolver, may not be fully realized until the new patterns are used throughout
a project.

# Alternatives

## The Module Normalization RFC

Perhaps the most prominent alternative that has been explored is the
[Module Normalization RFC](https://github.com/emberjs/rfcs/pull/124). Module
Unification shares many aspects with Module Normalization, but with one
fundamental difference: buckets in Module Normalization are normalized away
for the resolver, while collections in Module Unification play an important
role in module resolution.

The Ember Core Team decided that the sleight of hand required to allow buckets
to be used for organization only, and not for resolution, could create
confusion. Essentially, modules could conflict across buckets, because they
could have matching namespaces, names, and types. This kind of conflict could
not be allowed, so developers would need to understand too much about the
resolution strategy to make it ergonomic.

## Other Alternatives

[A large number of other alternatives have been explored](https://gist.github.com/dgeb/396fed953184acb04f4f)
before settling on this recommendation. Feel free to explore the history of any
of the linked gists to understand some of the subtle alternatives.

Of course, one alternative is to simply not change anything and accept the
drawbacks discussed in the Motivation section above. However, even if we accept
inefficiencies in our resolver and confusion over divergent file structuring
strategies, we still need to solve the "local lookup" problem, which does not
have a clean solution in today's module system.

# Unresolved questions

## How should tests be co-located in `src`?

Should tests be allowed within `src` via `*-test` types (e.g.
`component-integration-test`, `component-unit-test`, etc.) within respective
collections?

If this RFC is approved, then Robert Jackson plans to adapt the
[Grand Testing Unification RFC](https://github.com/emberjs/rfcs/pull/119) to
propose answers to these questions.

## What about routable components?

Should routable components have a type that's unique from other components?

Should they exist alongside `route` and `template` types in the `routes`
collection?

It seems plausible that routable components could simply use the `component`
type, and that we could lint against allowing template-invocable components
alongside routes.

## How should configuration declarations be made in the `main` module?

For example:

* How should resolvable exports be declared from addons?
* Can apps override the root names of addons? For example, if
  `ember-power-select` has a root name of `power-select`, could a consuming app
  override this?
* How do addons and apps declare their collection and type exports? For example,
  how could liquid-fire allow for a `transition` type and an `animations`
  collection?

## Should we allow collection groups?

Do the organizational benefits of collection groups outweigh the potential
confusion over where lines are drawn between a group/collection/namespace
when viewing a project structure.
