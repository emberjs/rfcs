---
Start Date: 2019-04-12
Relevant Team(s): Ember.js, Ember CLI
RFC PR: https://github.com/emberjs/rfcs/pull/481
Tracking: (leave this empty)

---

# Component Templates Co-location

## Summary

We propose to place a component's JavaScript file (the class) and its template
in the same directory on the file system.

Today:

```
app
├── components
│   ├── just-class.js
│   ├── my-widget.js
│   ├── nested
│   │   └── another-widget.js
│   └── ...
├── models
│   └── ...
├── routes
│   └── ...
├── services
│   └── ...
├── templates
│   ├── components
│   │   ├── just-template.hbs
│   │   ├── my-widget.hbs
│   │   ├── nested
│   │   │   └── another-widget.hbs
│   │   └── ...
│   ├── some-route.hbs
│   └── ...
└── ...
```

Proposed:

```
app
├── components
│   ├── just-class.js
│   ├── just-template.hbs
│   ├── my-widget.hbs
│   ├── my-widget.js
│   ├── nested
│   │   ├── another-widget.hbs
│   │   └── another-widget.js
│   └── ...
├── models
│   └── ...
├── routes
│   └── ...
├── services
│   └── ...
├── templates
│   ├── some-route.hbs
│   └── ...
└── ...
```

## Motivation

Today, a component's JavaScript file is located in the `app/components`
directory. On the other hand, its template is located in the
`app/templates/components` directory.

This design decision is a relic of Ember's pre-1.0 days. In those early days,
before components were even conceived, "views" were the primary building blocks
in Ember's UI programming model. Like components, they consist of a view class
located in `app/views/$NAME.js` (subclassing `Ember.View`), and/or a template
located in `app/templates/$NAME.hbs`. In other words, every template in Ember
could optionally have an associated view class, and vice versa.

Similar to components, they can be "invoked" in other templates using either
the `{{view "$NAME"}}` or `{{#view "$NAME"}}...{{/view}}` syntax. At the time,
this was how Ember developers create reusable pieces of UI in their app.

Unlike components, when a view is "invoked", the view instance does not become
the `this` context of its template. Instead, the `this` from the calling side
will persist (similar to partials), while properties on the view instance are
accessible from within the template with `{{view.someProperty}}` syntax.

Over time, this programming model has gone through several iterations of
refinements (remember the short-lived "controls"? 😀), eventually evolved into
the component-based programming model as we know it today.

When components were introduced, it was decided that they would live in a
separate "namespace" than views: their JavaScript files are located in
`app/components` instead of `app/views`, and it uses the now-familiar curly
invocation syntax instead of the `view` keyword.

This decision causes two problems. First, given a template file, such as
`app/templates/foo-bar.hbs`, how do we know if it belongs to a component or a
view? Furthermore, since they live in different namespaces, it is totally legal
to have unrelated components and views with the same names, which could result
in naming collisions if their templates were both located in `app/templates`.
To solve this problem, component templates were also given a separate namespace
`app/templates/components`, as opposed to sharing `app/templates` with views.

Over the course of the Ember 1.x release series, components have taken over as
the primary UI programming primitive. Views were eventually deprecated and
removed from Ember's API entirely in Ember 2.0. Despite that, the ramifications
of that era are still around us.

We argue that the current file system layout for component files is one such
example. Given how much Ember has evolved since its early days, most of the
design constraints we had at the time no longer apply. On the other hand, new
features in the framework – such as the introduction of Ember CLI and the addon
ecosystem – has brought new capabilities and constraints to the table. As a
result, this design has caused a lot of "ergonomics papercuts", which we have
enumerated below. While each of these may feel inconsequential on their own,
together they cause a lot of unnecessary friction and feel out-of-place in the
modern Ember experience.

### 1. Component class/template coupling

The JavaScript class and template of a component are not just _related_ but
fundamentally _coupled_. Together, they provide the whole implementation of
the component. It is impossible to understand one of them without the other.
When one file is changed, the other likely need to be updated as well. Placing
these files physically far apart on the file system fails to communicate this
tight coupling, in addition to causing a lot of inconveniences when navigating
a project.

### 2. No single enumeration of components

Given the importance of components in modern Ember apps, when navigating a
large-scale or otherwise unfamiliar project, some of the most common task for
Ember developers is to enumerate all components from the given app or to
determine whether there exists a component with a specific name. The former is
useful for getting a general sense of the scope, scale and coding style, and
the latter is sometimes necessary to disambiguate components from helpers or
to determine whether a component is part of the app or provided by an addon.

Unfortunately, under the current filesystem layout, neither `app/components` or
`app/templates/components` provide a complete view for those purposes. With
template-only components, it is possible to define a component without a class,
therefore a casual scan of `app/components` will not discover those components.
With the release of Octane, we expect template-only components to become more
common. Likewise, albeit less common, it is also possible to define a component
with _only_ a class, making `app/templates/components` potentially incomplete
also.

Therefore, the "correct" way to explore the components in an app is to mentally
merge the contents of both folders, which is difficult and counterproductive.

### 3. Deviation from route template conventions

In the view-centric days of Ember, `app/templates` was a "kitchen-sink" folder
without much internal structure – it mixes route templates with reusable view
templates, as well as component templates, each of which could be grouped into
their own deeper nesting structure.

With Ember 2.0 removing views, and therefore view templates, `app/templates`
now has a clear purpose and internal structure: each template file corresponds
to a route, and each folder corresponding to a nested route. For example,
`app/templates/post.hbs` is the template for the top-level `post` route, and
`app/templates/admin/authors/new.hbs` is the template for `admin.authors.new`.

Of course, there is one prominent exception to this: `app/templates/components`
breaks the otherwise clear convention. Unlike other folders in `app/templates`,
`app/templates/components` does not represent a nested route. Furthermore, its
internal structure (if any) also differs from its siblings: folders represent
arbitrary logical/domain groupings, rather than sub-routes.

Since `app/templates/components` did not start with a special character (e.g.
`app/templates/-components`), it shows up in the middle between its routes
siblings in most apps, due to lexicographical sorting of the filenames.

This inconsistency causes confusion and prevents us from teaching the otherwise
clear convention crisply.

### 4. Lack of single import

Even though the JavaScript class and template of a component are fundamentally
coupled and inseparable pieces, this relationship is only very loosely encoded
in Ember today. In fact – the only thing that ties them together is their name:
when Ember tries to render a component, it will separately lookup the class and
template by name through the resolver. These elements can both be present at
the same time, or in the cases of template-only or class-only components, one
of the resolutions will return a negative result. Only when both elements are
missing would Ember declare the component missing and raise an error.

This setup falls short when designing components meant for subclassing. For
example:

```js
// app/components/foo-bar.js

import MyParentComponent from "./my-parent";

export default class FooBarComponent extends MyParentComponent {
  // ...
}
```

As far as Ember can tell, this file defines a class-only component by the name
of `foo-bar`. The fact that it inherits from `my-parent` is an irrelevant and
unobservable detail. While the developer may expect this to also inherit the
parent component's template (`app/templates/components/my-parent.hbs`), Ember
has no way to know that the two are related.

Because of this problem, addons typically use the `layout` property from the
classic component API to get around this problem. When generating a component
in an addon, this is the default output from the blueprint:

```js
// addon/components/foo-bar.js

import Component from "@ember/component";
import layout from "../templates/components/foo-bar";

export default Component.extend({
  layout
});
```

```hbs
{{!-- addon/templates/components/foo-bar.hbs --}}

{{yield}}
```

```js
// app/components/foo-bar.js

export { default } from "my-addon/components/foo-bar";
```

First, within the `addon` folder, there is the component class and template.
Having these files in the `addon` folder allow them to be imported from
JavaScript (e.g. `import FooBar from 'my-addon/components/foo-bar';`) for
subclassing. However, as mentioned above, this does not also bring along the
component's template.

To avoid developers having to also import the template, the addon component
blueprint sets the `layout` property on the component's class. At runtime,
when Ember fail to find a template associated with the component, it will
then fall back to this property.

Finally, it re-exports the component in the `app` folder, which is merged with
the app's `app` folder, allowing the component to be resolved by the resolver
and invoked from Handlebars.

This workaround gets the job done, but has several major drawbacks.

First, it adds a significant amount of boilerplate to addon components, which
could be puzzling even to seasoned addon developers. In fact, the problem it
solves is unique to addons – there is nothing wrong with using inheritance in
apps – we just didn't think we could justify adding this amount of boilerplate
to every app and putting them in Ember's "happy path".

Second, the `layout` property in classic component uses a feature in Glimmer VM
called "late-bound layout", in which we must assume a different template for
each _instance_ of the component (the `layout` property can be set in `init`).
This is fundamentally more difficult to optimize and excludes the component
from key optimizations (and opportunities for future optimizations).

Most importantly, Glimmer components do not have the equivalent of the `layout`
property, meaning that addons (or components designed for inheritance in mind)
cannot be written using Glimmer components today.

To solve these problems once and for all, we need to change things such that
component templates are automatically associated with their JavaScript class at
build time, and importing the JavaScript class should result in a value that
has this metadata associated with it, which needs to survive subclassing as
well. Solving this problem is also essential for unblocking template imports.

We believe the technical design for allowing co-location will solve this
problem nicely.

## Detailed design

### High-level design

We propose to allow placing a component's template adjacent to its JavaScript
file in `app/components`. For example, for a component named `foo-bar`, it will
be `app/components/foo-bar.js` and `app/components/foo-bar.hbs`.

In addition, per the node resolution convention, we propose to allow `index`
files inside a directory to have the equivalent semantics. In the example
above, it could also be structured as `app/components/foo-bar/index.js` and
`app/components/foo-bar/index.hbs`. This allows additional files related to the
component (such as a `README.md` file) to be co-located on the filesystem.

For template-only components, they can be either `app/components/foo-bar.hbs`
or `app/components/foo-bar/index.hbs` without a corresponding JavaScript file.

Similarly, for addons, templates can be placed inside `addon/components` with
the same rules laid out above.

In all of these case, if a template file is present in `app/components` or
`addon/components`, it will take precedence over any corresponding template
files in `app/templates`, the `layout` property on classic components, or a
template with the same name that is made available with the resolver API.
Instead of being resolved at runtime, a template in `app/components` will be
associated with the component's JavaScript class at build time.

### Low-level primitives

We propose to introduce the following low-level APIs:

- The `setComponentTemplate` function takes two arguments, the first being the
  pre-compiled (wire-format) template, the second being the component class. It
  transparently associates the given template with the component class in a way
  can be retrieved later with the `getComponentTemplate` function described
  below. For convenience, `setComponentTemplate` will return the component
  class (the second argument).

- The `getComponentTemplate` function takes a component class and returns the
  template associated with the given component class, if any, or one of its
  superclasses, if any, or `undefined` if no template association was found.

This is one possible way to implement these functions:

```js
function setComponentTemplate(template, componentClass) {
  Object.defineProperty(componentClass, "__template__", {
    configurable: true,
    enumerable: false,
    writable: false,
    value: template
  });

  return componentClass;
}

function getComponentTemplate(componentClass) {
  return componentClass.__template__;
}

class Foo {}
class Bar extends Foo {}
class Baz extends Bar {}

class Bat {}

// USAGE

setComponentTemplate(template("foo template"), Foo);
setComponentTemplate(template("baz template"), Baz);

getComponentTemplate(Foo); // => foo template
getComponentTemplate(Bar); // => foo template (inherited)
getComponentTemplate(Baz); // => baz template (overridden)

getComponentTemplate(Bat); // => undefined
```

In an actual implementation, we would probably want to avoid polluting the
component class with a string key (`__template__` in this example) and use a
`Symbol` or `WeakMap` based strategery.

For performance reason, changing the template is not allowed once set. It is
also illegal to call `setComponentTemplate` on a component class that has
already been rendered, or once `getComponentTemplate` has been called on it.
Together, these rules ensure the results from calling `getComponentTemplate`
can be reliably cached, either by the internals of `getComponentTemplate`
itself or by one of its callers.

While the default export of a component's JavaScript file is usually a class,
it is not a strict requirement with custom component managers. To accommodate
this, `setComponentTemplate` can be passed any JavaScript `Object`.

### Build-time transformations

The low-level `setComponentTemplate` and `getComponentTemplate` APIs are not
intended to be called by end-users directly, even though they will be public
and part of our semver stability guarantee. Instead, they are meant to be used
primarily in the output emitted by build tools. At build time, any template
files found in `{app,addon}/components` will be inlined into the component's
JavaScript file and removed from the build output. For example, given these
files on disk:

```js
// app/components/foo-bar.js

import Component from "@ember/component";

export default Component.extend({
  // ...
});
```

```hbs
{{!-- app/components/foo-bar.hbs --}}

foo bar!
```

The build output will be something to the effect of this JavaScript file:

```js
// app/components/foo-bar.js

import Component, { setComponentTemplate } from "@ember/component";

// output of compiling "foo bar!" with ember-cli-htmlbars
const TEMPLATE = Ember.HTMLBars.template({
  id: "...",
  block: "...",
  meta: { moduleName: "app/components/foo-bar" }
});

const CLASS = Component.extend({
  // ...
});

export default setComponentTemplate(TEMPLATE, CLASS);
```

The variables are named here for clarity, but the actual build output would
be careful to avoid introducing hygiene issues and other observable semantic
changes to the JavaScript file.

One caveat here is that each component JavaScript file should export a value
that is unique to that file. For example, this should be **avoided**:

```js
// app/components/foo-bar.js

import MyParentComponent from "./my-parent";

// BAD: don't do this!
export default MyParentComponent;
```

This is problematic because `setComponentTemplate` will be called on
`MyParentComponent` directly, affecting the parent component and all of its
descendants. This can be avoided by subclassing, even when no customization is
required:

```js
// app/components/foo-bar.js

import MyParentComponent from "./my-parent";

// GOOD: do this instead!
export default class extends MyParentComponent {}
```

Most cases of this problem can be linted against easily.

### Template-only components

For template-only components, we propose to introduce the following low-level
API:

- The `templateOnlyComponent` function takes no arguments and produces a unique
  value that can be used to represent a template-only component.

Again, this function, though public, is not intended to be called by users
directly. It is primarily used in the output emitted by build tools. When a
template is found in `{app,addon}/components` but without a corresponding
JavaScript file, the build output will be something similar to the following:

```js
// app/components/foo-bar.js

import Component, { setComponentTemplate } from "@ember/component";
import templateOnlyComponent from "@ember/component/template-only";

// output of compiling "foo bar!" with ember-cli-htmlbars
const TEMPLATE = Ember.HTMLBars.template({
  id: "...",
  block: "...",
  meta: { moduleName: "app/components/foo-bar" }
});

const CLASS = templateOnlyComponent();

export default setComponentTemplate(TEMPLATE, CLASS);
```

In addition to build tooling, addon authors may also find this function
useful. Currently, the only way to create a "true" template-only component
is by enabling an optional feature in the app. Since addons cannot assume the
value of that flag in the consuming app, they currently cannot take advantage
of the feature. By providing this function, addon authors can work around this
problem by explicitly defining a JavaScript file with `templateOnlyComponent()`
as the default export.

On the other hand, app developers, should _not_ have any reason to use this
function directly in their app, since they could just as easily enable the
optional feature across the board.

### Codemod

A codemod will be provided to seamlessly migrate component templates.

Such a "codemod" will essentially just merges `app/templates/components` into
`app/components`. For a quick "taste test" of what the resulting tree will look
like in your app, the following commands give a close approximation:

```bash
$ rsync --archive --remove-source-files app/templates/components/ app/components/
$ rm -rf app/templates/components/
```

Of course, the resulting output won't "work", but it can be useful for getting
a sense of what it's like to work with the new layout on editors, Github, etc.

### Generator

We propose to make some updates to the components generator to accompany this
change.

1. It should accept a `--component-class` option. This controls which base
   class is used for that component and whether native classes are used. The
   legal values for this option are `@glimmer/component` (aliased as `-gc`),
   `@ember/component` (aliased as `-cc`), `@ember/component/template-only`
   (aliased as `-tc`) or an empty string (aliased as `--no-component-class` and
   `-nc`).

   The latter two differ in that `@ember/component/template-only` would
   generate an explicit JavaScript file with `templateOnlyComponent()` as the
   default export, which is useful for addons, whereas `--no-component-class`
   would skip generating a JavaScript file altogether.

   This option may be extended in the future to allow other custom components
   to provide their own blueprints, but for now, passing anything other than
   the allowed values will result in an error.

2. It should accept a `--component-structure` option.

   When this option is set to `flat` (aliased as `-fs`), the component's JavaScript and template files will both be generated at the root of
   `{app,addon}/components`.

   When this option is set to `nested` (aliased as `-ns`), a folder will be
   generated for the component in `{app,addon}/components`, and the component's
   JavaScript and template files will be generated as `index.{js,hbs}` inside
   the folder.

   When this option is set to `classic` (aliased as `-cs`), the component's
   template file will be generated in `{app,addon}/templates/components`. For
   addons, when used with `--component-class=@ember/component`, this will also
   emit the `layout` property workaround.

   When this option is set to `pods` (aliased as `--pods`), it will generate
   `{app,addon}/components/$name/{component.js,template.hbs}`.

3. Thses options will default to `--component-class=@ember/component` and
   `--component-structure=classic` for backwards compatibility.

   However, the default values can be overridden in `.ember-cli` as usual, and
   teams are encouraged to do so as they see fit. Due to a limitation in how
   the system works, the names for these options are chosen such that they are
   unlikely to conflict with options on other `ember` commands, which is why
   they are a bit verbose.

   For Octane apps, the default app blueprint will include a `.ember-cli` file
   that defaults to `--no-component-class` and `--component-structure=flat`.
   The guides and documentation will assume these settings going forward.

## How we teach this

As mentioned above, we will update the learning resources to assume the "flat"
co-located layout. Throughout the Ember Guides, the Tutorials, and the CLI
Addon Tutorial, we would update the file paths in all component examples. Since
we assume the classic layout today, in most cases only the template paths would
need to be updated. The prose describing the location of files would also need
to change.

The API documentation will describe the full set of file layout options
supported in the major version of Ember. A section should also be added to the
CLI guides, which describes how to properly import components from addons that
have a mix of layout types. We will not cover blending file layouts in the
Ember Guides, since they represent the happy path for an app, but we could link
to the CLI Guides explanation.

With this layout, it should be much easier for new users to form a mental model
around components. We can start by teaching that the most basic component is
just a reusable piece of markup (template-only components), but can "upgraded"
to have dynamic content by taking arguments using the `@name` syntax, and
further "upgraded" to keep internal states by creating a JavaScript file next
to the template and finally add interactivity with element modifiers.

Because template-only component is such a light-weight concept, it is arguably
not necessary to separate out the topic of "templates" from "components" in the
guides, but this can be addressed in a future revision of the guides.

We will teach that the `templates` folder is used for route templates. This can
be introduced in the Guides at the same time as the `routes` and `controllers`
folders/topics, which come later in the Table of Contents.

## Drawbacks

Another drawback is that it only address the co-location issue for components,
not other related types like [[route, controller, route template]] and [[model,
adapter, serializer]], or even co-location of tests. However, we believe the
situation with components is unique enough (see the motivation section) that
they are not merely _related_, but _coupled_. That, along with the fact that
components are much more common, sets them apart from the rest and justifies
solving the problem first.

There is a small risk that we will subject the community to another migration
when we finalize the replacement of Module Unification (the "New File Layout").
However, we feel pretty confident that regardless of where the collection of
components will end up on disk (e.g. `src/ui/components`), the internal
structure of that collection will closely match what is proposed in this RFC.
Ultimately, we expect there to be automatic migrators for these kinds of
changes anyway, so the cost of the possible churn is contained.
