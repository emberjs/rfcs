- Start Date: 2016-03-07
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

Create a single normalized module naming pattern coupled with new ergonomic file
naming patterns.

# Motivation

Ember has always succeeded because of its strong conventions, which improve the
learning curve for new developers and make it easy for experienced developers to
join Ember projects that are already underway. Strong conventions also allow the
Ember community to rally around a pattern to shape a single, well done
implementation.

Of course, it's important to occasionally question Ember's conventions to ensure
that they are working well for the community. Conventions must evolve to support
new capabilities in the framework and JavaScript in general. On the other hand,
changes shouldn't be considered lightly (hence this RFC process).

File naming and module resolution in Ember CLI projects deserve a rethink. The
current system is more complex than necessary, which both steepens the learning
curve and limits its technical potential.

Drawbacks include:

* Confusion over which of two orthogonal approaches to use for organizing
  modules:

  * classic - modules are organized at the top-level by "type"
    (`components`, `templates`, etc.) and then by namespace and name.

  * pods - modules are organized by namespace, then name, then type.

* Multiple top-level directories and special rules for addons:

  * Depending on whether you're developing an addon or an application, files
    should be placed in either the `addon` or `app` directory.

  * Addons can additionally define modules to be merged into an application
    through their own `app` directory. These public modules are typically
    private modules that are imported and re-exported, which introduces an extra
    module per export and is confusing for beginners.

* Because addons' modules are mixed into an application, there's the possibility
  of naming collisions between two addons or an addon and its consuming
  application.

* Without a single canonical module structure, modules don't have a clear sense
  of "locality", which prevents the ability to declare modules that are
  available only in a "local" namespace (this as-yet unsupported feature has
  been called "local lookup").

* Inefficient module resolution due to the number of potential places to lookup
  a particular module by name.

Recognizing these drawbacks, the Core Team compiled a set of
[design constraints](https://github.com/emberjs/core-notes/blob/master/ember.js/2016-01/january-22.md#summary-of-pods-design-constraints)
for a rethink of Ember's approach to modules:

1. Reasonable branching factor. Users should see a reasonable number of items at any given level in their hierarchy. Flattening out too much results in an unreasonably large number of items.
2. No slashes in component names. The existing system allows this, but we don't want to support invocation of nested components in Glimmer.
3. Addons need to participate in the naming scheme, most likely with namespace prefix and double-colon separator.
4. Subtrees should be relocatable. If you move a directory to a new place in the tree, its internal structure should all still work.
5. There should be no cognitive overhead when adding a new thing. The right way should be obvious and not impose unnecessary decisions.
6. We need clean separation between the namespace of the user's own components, helpers, routes, etc and the framework's own type names ("component", "helper", etc) so that we can disambiguate them and add future ones.
7. Ideally we will have a place to put tests and styles alongside corresponding components.
8. Local relative lookup for components and helpers needs to work.
9. Avoid the "titlebar problem", in which many files are all named "component.js" and you can't tell them apart in your editor.

This proposal attempts to address these constraints with a single consistent
approach to modules that will make Ember easier to use and learn _and_ improve
the efficiency of Ember's resolver at run-time.

# Detailed design

Implementation will involve a new normalized naming pattern for modules and a
simple process for module resolution.

Module files will be organized in a new `modules` directory that can co-exist
alongside the `app` and/or `addon` directories (until they are deprecated).

## Normalized modules

This proposal introduces the concept of module "normalization", in which
all of an Ember project's modules can be mapped to a single normalized form.
The Ember CLI resolver will exclusively lookup modules based upon this
normalized form.

Every normalized module path consists of the following parts:

* `namespace` - zero or more segments for grouping modules (e.g. `posts/new`)
* `name` - the specific name of a module (e.g. `post-editor`)
* `type` - the base class of a module (e.g. `component`, `template`,
           `helper`, etc.)

All normalized module paths will be formed with the following pattern:

```
${namespace}/${name}/${type}
```

### Normalized modules for an example application

All of an application's normalized modules will be structured within a single
tree of namespaces.

A simple application might include the following modules:

```
auth/instance-initializer
auth/service
capitalize/helper
comments/adapter
comments/model
comments/serializer
i18n/initializer
list-paginator/component
list-paginator/template
main/app
main/router
main/template
posts/adapter
posts/model
posts/serializer
posts/new/new-post/component
posts/new/new-post/template
posts/post-editor/post-editor-button/component
posts/post-editor/post-editor-button/template
posts/post-editor/calculate-post-title/helper
posts/capitalize/helper
posts/new/route
posts/new/template
posts/post-editor/component
posts/post-editor/template
posts/post-viewer/component
posts/post-viewer/template
posts/titleize/helper
posts/route
posts/template
```

Note that every module needs a `name` and `type`. Top-level modules like
`app` and `router` have been assigned the name `main`.

Don't worry about the ergonomics of module names and paths. The following
section describes how a set of file and directory naming rules can be used to
distill an ergonomic file structure down to the above module structure.

## Directory and file naming

### Modules directory

A new top-level directory will be introduced to Ember CLI: `/modules`. This
directory will be used to contain the files that represent all the modules in
an application or addon.

### Base file naming pattern

The base file naming pattern aligns directly with the normalized module naming
pattern described above. File paths will be formed with the following pattern:

```
modules/${namespace}/${name}/${type}
```

The example normalized modules above could be derived _directly_ from the
following files and directories:

```
modules
  posts
    new
      new-post
        component.js
        template.hbs
      route.js
      template.hbs
    post-editor
      post-editor-button
        component.js
        template.hbs
      calculate-post-title
        helper.js
      component.js
      template.hbs
    post-viewer
      component.js
      template.hbs
    capitalize
      helper.js
    titleize
      helper.js
    route.js
    template.hbs
    adapter.js
    model.js
    serializer.js
  auth
    instance-initializer.js
    service.js
  capitalize
    helper.js
  comments
    adapter.js
    model.js
    serializer.js
  i18n
    initializer.js
  list-paginator
    component.js
    template.hbs
  main
    app.js
    router.js
    template.hbs
```

Although this structure is straightforward, it is not particularly ergonomic
to work with for many large applications.

### Alternative file naming patterns

It's important to recognize the value of the orthogonal naming systems currently
allowed by Ember CLI at the same time we recognize the problems they cause.

The value is that they allow for two different forms of organization: by type
and by function. Depending on the module, one or the other may fit more
naturally.

On the other hand, problems are caused because the current system creates two
mutually incompatible module structures. This makes module resolution less
efficient and prevents features like local lookup from working.

It is possible to combine most of the value of the current system while solving
its problems by allowing flexibility in _file_ naming while ensuring consistency
in _module_ naming.

#### Allow dot-separated `name.type` files

It's proposed that we allow the following alternative file naming pattern which
combines a module's `name` and `type` into a single file name:

* `modules/${namespace}/${name}.${type}` - dot-separated `name` and `type`

This is an especially useful pattern when a module's `name` is unlikely to form
a child namespace (e.g. it's unlikely that you'll want to form the `capitalize`
namespace from `capitalize.helper.js`).

Our example app's modules could alternatively be created with the following
files:

```
modules
  posts
    new
      new-post.component.js
      new-post.template.hbs
    post-editor
      post-editor-button.component.js
      post-editor-button.template.hbs
      calculate-post-title.helper.js
    capitalize.helper.js
    new.route.js
    new.template.hbs
    post-editor.component.js
    post-editor.template.hbs
    post-viewer.component.js
    post-viewer.template.hbs
    titleize.helper.js
  main.app.js
  main.router.js
  main.template.hbs
  auth.service.js
  auth.instance-initializer.js
  capitalize.helper.js
  comments.adapter.js
  comments.model.js
  comments.serializer.js
  i18n.initializer.js
  list-paginator.component.js
  list-paginator.template.hbs
  posts.adapter.js
  posts.model.js
  posts.route.js
  posts.serializer.js
  posts.template.hbs
```

Note that this naming pattern can be intermingled with the base pattern without
conflict.

#### Simplify "main" modules

One more rule is proposed to simplify file naming: top-level modules
named `main` can omit their name and just be named by `type` without
ambiguity:

* `modules/${type}` - `name` is assumed to be `main` when omitted

Instead of `modules/main/app.js` or `modules/main.app.js`, the
simpler alternative `modules/app.js` can be allowed.

#### Organize files in "buckets"

One last optimization is to reduce the noise caused by directories filled with
files of all different types.

It's proposed that we introduce special "bucket" directories for organizing
files. These directories should be prefixed with a `-` to indicate that they
will not be used to inform module names, namespaces, or types. Every module
defined within a bucket simply flows through to the bucket's parent namespace.

Ember CLI should probably enforce the use of these buckets as well as the types
of modules allowed in each bucket.

The following buckets are proposed:

* `-components` - components and their corresponding templates
* `-data` - including models, adapters, and serializers
* `-helpers` - helpers
* `-initializers` - application and instance initializers
* `-routes` - routes, templates, and routable components
* `-services` - services

### Files for an example application

Given the base and alternative _file_ naming patterns, the example application
could be restructured using the following files and directories:

```
modules
  -routes
    posts
      new
        -components
          new-post
            component.js
            template.hbs
        route.js
        template.hbs
      -components
        post-editor
          -helpers
            calculate-post-title.helper.js
          post-editor-button
            component.js
            template.hbs
          component.js
          template.hbs
        post-viewer
          component.js
          template.hbs
      -helpers
        capitalize.helper.js
        titleize.helper.js
      route.js
      template.hbs
  -components
    list-paginator
      component.js
      template.hbs
  -data
    comments
      adapter.js
      model.js
      serializer.js
    posts
      adapter.js
      model.js
      serializer.js
  -helpers
    capitalize.helper.js
  -initializers
    auth.instance-initializer.js
    i18n.initializer.js
  -services
    auth.service.js
  app.js
  router.js
  template.hbs
```

This example shows that we can improve the ergonomics of naming _files_ in
Ember CLI projects, while normalizing everything into a single _module_
naming structure.

## Normalized module:file map

The Ember CLI resolver will need to maintain a map of normalized module
names to file locations on disk. This map will provide a single place for
module lookups by the resolver.

Construction of this map will allow for detection of any module name conflicts.
In the case of any conflicts, Ember CLI should throw an exception.

## Addon modules

An addon's modules will no longer be "mixed in" to a consuming application
directly. Instead, the addon's modules will be accessible at a namespace.
Addons will be able to publish a default namespace (e.g. `ember-power-select`
might choose `power-select`), which consuming applications will be able to
alias to avoid conflicts.

Private modules will exist alongside public modules in addons within a
`-private` namespace. This namespace will provide a strong signal to consumers
that private modules should not be accessed.

### Namespaced components and helpers

A `::` separator will be used to namespace components and helpers from
templates.

If an addon namespaced as `magic-ui` has a root-level `select/component`
module, then that component can be rendered as `{{magic-ui::select}}`.

Top-level component and helper modules named `main` can be accessed
directly at the addon's namespace. If `ember-power-select` uses a top-level
namespace of `power-select` and has a root-level `main/component` module, then
that component can be rendered simply as `{{power-select}}`.

## Local lookups

Local lookup requires that module resolution occurs first "locally" before
happening at the "global" (or top) level. This will improve the utility of
components and helpers localized to an area of an application.

Local lookups are made simple when there's only one place to look "locally" -
in a namespace defined by the name of the current template.

Say that the template `modules/posts/post-editor/template` references the
`{{post-editor-button}}` component. This module could be found "locally" in
`modules/posts/post-editor/post-editor-button/component`.

## Co-existence with current module and file structures

It will be necessary to allow a project to migrate its files from the current
`app` and / or `addon` directories to the new `modules` directory. This will
require modifications to the
[ember resolver](https://github.com/ember-cli/ember-resolver).

# How We Teach This

The Ember guides will need to be updated significantly to reflect the new
conventions.

We need to teach the new normalized module structure by way of the new file
structure, since there is a direct mapping between the two. This mapping is made
clear through the very name of the `modules` folder. The file structure should
be taught as a distillation of the best elements of the "classic" and "pods"
structures.

Developers should understand that every file path and name maps to a module's
`namespace`, `name`, and `type`, and that these elements are always explicitly
declared. This will clarify how we can allow both the `name/type` and
`name.type` forms. Developers will see how nested namespaces like routes
naturally favor the `name/type` pattern, in which each `name` segment can become
part of the `namespace` for child modules. And they will see how non-nested
modules like helpers are more clearly named with the `name.type` pattern. We can
teach buckets by showing how a top-level `-data` bucket naturally isolates data
concerns from UI concerns.

We have the benefit of using Ember CLI's generators to get developers started
and keep them on track as they add new modules. Although we can allow any of the
base and alternative file naming patterns to coexist, we still need to decide on
a base pattern that's used by blueprints. I'd recommend that we use top-level
buckets for grouping modules.

A tool such as Ember Watson could be enhanced to automatically translate
projects that use either the classic or pods structure into the new modules
structure.

Furthermore, the Ember Inspector could be enhanced to understand the new
modules structure.

# Drawbacks

Any change to such a fundamental pattern as file naming will incur some
mental friction for developers who are accustomed to the current conventions.
It is hoped that tooling like Ember Watson can lessen this friction by
automating transitions.

Although we need some way to prevent confusion between buckets with module
namespaces, some developers may find the `-` bucket prefix aesthetically
unappealing. Furthermore, it may be easy to forget to include this prefix when
creating modules without generators.

Another adjustment developers will need to make is to ensure that each module's
`type` is explicitly represented in its file path. This will be more natural
for users of pods, who have been doing this already, but less natural for users
of the classic structure who have counted on top-level buckets to convey `type`.

Of course, we won't prevent usage of the currently used patterns for some time,
but they will eventually be deprecated. Some efficiencies won't be fully
realized until the new patterns are used throughout an application.

# Alternatives

[A large number of alternatives have been explored](https://gist.github.com/dgeb/396fed953184acb04f4f)
before settling on this recommendation. Feel free to explore the history of any
of the linked gists to understand some of the subtle alternatives.

One alternative is to simply not change anything and accept the drawbacks
discussed in the Motivation section above. However, even if we accept
inefficiencies in our resolver and confusion over divergent file structuring
strategies, we still need to solve the "local lookup" problem, which does not
have a clean solution in today's module system.

# Unresolved questions

* What specific file naming pattern should be used by default for each type
  of blueprint?

* Should we tighten the flexibility of the alternative patterns? For example,
  we could require top-level buckets or emit warnings when certain naming
  patterns are used with certain types.

* Should a prefix other than `-` be considered for buckets? We need some kind
  of reserved prefix / marker to indicate that buckets are not namespaces.

* The `-routes` bucket contains routes, templates, and (eventually) routable
  components. Should this bucket be renamed to something more general, since
  it obviously contains more than just "routes"? Or should we simply not
  encourage any top-level bucket for these modules, which represent the core
  structure of most applications?

* Should routable components be given a `type` that's unique, or should they
  also use `component`? This might be a question best left on the back-burner
  for now, considering routable components have not yet landed.

* Should we allow module types for tests, such as `component-test`, so that
  test modules can be placed alongside application code? Should we also
  introduce an optional `-tests` bucket for further separation?
