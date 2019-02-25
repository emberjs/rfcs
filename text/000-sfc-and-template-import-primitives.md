- Start Date: 2019-02-22
- Relevant Team(s): Ember.js
- RFC PR: https://github.com/emberjs/rfcs/pull/454
- Tracking: (leave this empty)

# SFC & Template Import Primitives

## Summary

Expose low-level primitives for associating templates with component classes
and customizing a template's ambient scope.

These primitives unlock experimentation, allowing addons to provide
highly-requested features (such as single-file components) via stable, public
API.

## Motivation

This proposal is intended to unlock experimentation around two
highly-requested features:

1. Single-file components.
2. Module imports in component templates.

Although exploring these features is the primary motivation, an important
benefit of stabilizing low-level APIs is that they enable the Ember community
to experiment with new, unexpected ideas.

### Single-File Components

In Ember components today, JavaScript code lives in a `.js` file and template
code lives in a separate `.hbs` file. Juggling between these two files adds
friction to the developer experience.

Template and JavaScript code are inherently coupled, and changes to one are
often accompanied by changes to the other. Separating them provides little
value in terms of improving reusability or composability.

Other component APIs eliminate this friction in different ways. React uses
JSX, which produces JavaScript values:

```js
// MyComponent.jsx

export default class extends Component {
  state = { name: 'World' };

  render() {
    return <div>Hello {this.state.name}!</div>
  }
}
```

Vue has an optional [`.vue` single-file component (SFC)
format](https://vuejs.org/v2/guide/single-file-components.html):

```html
<!-- MyComponent.vue -->

<template>
  <div>Hello {{name}}!</div>
</template>

<script>
export default {
  data() {
    return {
      name: 'World'
    }
  }
}
</script>
```

### Module Imports in Templates

One benefit of JSX is that it leverages JavaScript's existing scope
system. React components are JavaScript values that can be imported and
referenced like any other JavaScript value. If a binding would be in scope
in JavaScript, it's in scope in JSX:

```js
import { Component } from 'react';
import OtherComponent from './OtherComponent';

export default class extends Component {
  render() {
    return (
      <div>
        // We know OtherComponent is in scope because it was imported at the top
        // of the file, so we can invoke it here as a component.
        <OtherComponent />
      </div>
    );
  }
}
```

This is a bit nicer than Vue's SFCs, where JavaScript code and template code
happen to be in the same file but otherwise interact no differently than if
they were still in separate files. Developers must learn a proprietary Vue
API for explicitly copying values from JavaScript's scope into the template's
scope, and component names can be different between where they are imported
(`OtherComponent`) and where they are invoked (`other-component`):

```html
<template>
  <div>
    <other-component />
  </div>
</template>

<script>
import OtherComponent from './OtherComponent.vue'

export default {
  components: {
    OtherComponent
  }
}
</script>
```

Neither of these approaches is exactly right for Ember. Ideally, we'd find a
way to "bend the curve" of tradeoffs: maintaining the performance benefits of
templates, while gaining productivity and learnability by having those
templates seamlessly participate in JavaScript's scoping rules.

### Unlocking Experimentation

Rather than deciding on the best syntax for single-file components and
template imports upfront, this RFC proposes new low-level APIs that addons
can use as compile targets to implement experimental file formats.

For example, imagine a file format that uses a frontmatter-like syntax to
combine template and JavaScript sections in a single file:

```hbs
---
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';
import titleize from './helpers/titleize';
import BlogPost from './components/blog-post';

export default class MyComponent extends Component {
  @service session;
}
---

{{#let this.session.currentUser as |user|}}
  <BlogPost @title={{titleize @model.title}} @body={{@model.body}} @author={{user}} />
{{/let}}
```

Or imagine this syntax, where a component's template is located syntactically
within the class body:

```js
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';
import titleize from './helpers/titleize';
import BlogPost from './components/blog-post';

export default class MyComponent extends Component {
  <template>
    {{#let this.session.currentUser as |user|}}
      <BlogPost @title={{titleize @model.title}} @body={{@model.body}} @author={{user}} />
    {{/let}}
  </template>
    
  @service session;
}
```

Or even a variation of the Vue SFC format:

```html
<template>
{{#let this.session.currentUser as |user|}}
  <BlogPost @title={{titleize @model.title}} @body={{@model.body}} @author={{user}} />
{{/let}}
</template>

<script type="module">
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';
import titleize from './helpers/titleize';
import BlogPost from './components/blog-post';

export default class MyComponent extends Component {
  @service session;
}
</script>
```

With the API proposed in this RFC, an Ember addon could transform any of the
above file formats, at build time, into a JavaScript file that looks
something like this:

```js
import Component from '@glimmer/component';
import { inject as service } from '@ember/service';
import titleize from './helpers/titleize';
import BlogPost from './components/blog-post';

import { createTemplateFactory, setComponentTemplate } from '@ember/template-factory';

const wireFormat$1 = {
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "meta": { "moduleName": "src/ui/components/MyComponent.js" },
  "scope": () => [BlogPost, titleize]
};

const template$1 = createTemplateFactory(wireFormat$1);

export default setComponentTemplate(template$1, class MyComponent extends Component {
  @service session;
});
```

Note that the file formats shown above are hypothetical and used as
illustrative examples only. This RFC does *not* propose a recommended file
format. Rather, the goal is to expose public JavaScript API for:

1. Compiling a template into a JavaScript wire format.
3. Associating a compiled template with its backing JavaScript class.
2. Specifying which values are in scope for a given template at runtime.

By focusing on primitives first, the community can iterate on different SFC
designs via addons, before settling on a default format that we'd incorporate
into the framework.

## Detailed design

### Module API

This RFC proposes the following new modules and exports:

Name | Description  
---------|----------
`import { precompile } from '@ember/template-compiler'` |  Compiles template source code into a JavaScript wire format.
`import { createTemplateFactory } from '@ember/template-factory'` | Creates a template factory wrapper around the wire format.
`import { setComponentTemplate } from '@ember/template-factory'` |  Associates a template factory with a JavaScript component class.

Detailed descriptions and rationales for these APIs are provided below.

### Template Wire Format

Browsers don't understand Ember templates natively. So how are these
templates compiled and turned into a format that can be run in the browser?

The Glimmer VM supports two modes for compiling templates: an Ahead-of-Time
(AOT) mode where templates are compiled into binary bytecode, and a
Just-in-Time (JIT) mode where templates are compiled into an intermediate
JSON format, with final bytecode compilation happening on demand in the browser
the first time a template is rendered.

Ember uses Glimmer's JIT mode, so templates are sent to the browser as an
optimized JSON data structure we call the _wire format_. The
`@glimmer/compiler` package provides a helper function called `precompile`
that turns raw template source code into this "pre-compiled" wire format.

```js
import { precompile } from '@glimmer/compiler';
const json = precompile(`<h1>{{this.firstName}}</h1>`);
```

There is some additional processing Ember does on top of the Glimmer VM
compiler to allow compiled templates to work. This RFC proposes exposing this
functionality as public API via a `precompile` function exported from the
`@ember/template-compiler` package.

```js
import { precompile } from '@ember/template-compiler';
const json = precompile(`<h1>{{this.firstName}}</h1>`);
```

The exact structure of the wire format returned from `precompile` is not
specified, is not considered public API, and is likely to change across minor
versions. Other than a guarantee that it is a string that can be embedded in
JavaScript and is a valid JavaScript expression, users should treat the value
returned by `precompile` as completely opaque.

### Template Factories

Once we have our template compiled into the wire format, we need a way to
annotate it so that Ember knows that the value represents a compiled
template. The wrapper object around the wire format that does this is called
a _template factory_.

This RFC proposes exposing a function for creating template factories called
`createTemplateFactory`, exported from the `@ember/template-factory` package.

Users would use this function to turn the raw wire format into a template
factory object at runtime:

```js
import { createTemplateFactory } from '@ember/template-factory';

const wireFormat = /* embed wire format from precompile here */
const templateFactory = createTemplateFactory(wireFormat);
```

### Associating Template Factories and Classes

In Ember, a component is made up of a template and an optional backing
JavaScript class. But how does Ember know which class and template go
together?

Today, the answer is "it's complicated." That complexity makes it difficult
for addons to override default behavior and experiment with things like
single-file components.

By default, templates are associated with a class implicitly by name. For
example, the template at `app/templates/components/user-avatar.hbs` will be
joined with the class exported from `app/components/user-avatar.js`, because
they share the base file name `user-avatar`.

The benefit of this approach is that files are in predictable locations. If
you are editing a template and need to switch to the class, or are reading a
another template and want to see the implementation of the `{{user-avatar}}`
component, finding the right file to jump to should not be difficult.

Sometimes, though, Ember developers run into scenarios where they'd like to
re-use the same template across multiple components. Ember supports
overriding a component's template by defining the `layout` property in the
component class:

```js
import Component from '@ember/component';
import OtherTemplate from 'app/templates/other-template';

export default Component.extend({
  layout: OtherTemplate
})
```

However, this capability comes at a cost. Supporting dynamic layouts like
this requires Ember to put the Glimmer VM into a mode where it checks each
component instance for a `layout` property, preventing these components from
being fully optimized.

This RFC proposes a more explicit, static system for associating templates
with component classes. A function called `setComponentTemplate` is exported
from the `@ember/template-factory` package. To create the association, this
function is invoked with the template factory and component class as
arguments:

```js
import Component from '@glimmer/component';
import { createTemplateFactory, setComponentTemplate } from '@ember/template-factory';

const wireFormat = /* wire format here */
const templateFactory = createTemplateFactory(wireFormat);

class MyComponent extends Component {
  /* ... */
}

setComponentTemplate(template, MyComponent);
```

For convenience, `setComponentTemplate` returns the same component class that
was passed, allowing the class to be defined and associated with a template
in a single expression:

```js
export default setComponentTemplate(template, class MyComponent extends Component {
  /* ... */
});
```

Even though this is a runtime API, template association is intended to be as
static and restricted as possible, allowing us to enable better optimizations
in the future:

* Templates should be set on component classes as early as possible, ideally
  immediately after the class is defined.
* A component's template cannot be set more than once. Attempting to change a
  component's template after one has been set will throw an exception.
* Setting a component's template after rendering has begun (even if no previous
  template was set) also throws an exception.
* Every instance of a given component has the same template. It is not
  possible to dynamically change a component's template at runtime.

Component subclasses inherit their template from the nearest superclass that
has had a template set via `setComponentTemplate`. If no superclass has an
associated template, Ember will fall back to the current template resolution
rules.

Templates set via `setComponentTemplate`, including those set on a
superclass, take precedence over the `layout` property, container lookup, or
any other existing mechanism of template resolution. For example, a template
set on a superclass via `setComponentTemplate` would take precedence over a
subclass's `layout` property (if it had one). However, a template set via
`setComponentTemplate` on the subclass would take precedence over the
superclass.

### Template Scope

In this context, _scope_ refers to the set of names that can be used to refer
to values, helpers, and other components from within your template. For
example, when you type `{{@firstName}}`, `{{this.count}}`, or `<UserAvatar
/>`, how does Ember know what each of these refers to? The answer to that is
determined by the template's scope.

Today, the names available inside a particular template are determined by a
few different rules:

1. `this` always refers to the component instance (if the template has a
   backing class).
2. Arguments (like `@firstName`) are placed into scope when you invoke the
   component.
3. Block parameters (like `item` in `{{#each @items as |item|}}`) are in
   scope, but only inside the block.
4. Anything else, like components and helper names, must be looked up
   dynamically through Ember's container. This lookup process is somewhat
   complex and not always as fast as we'd like.

Because the rules for how components and helpers are looked up are implicit
and can be confusing for new learners, we'd like to explore alternate APIs
that allow users to explicitly import values and refer to them from
templates.

This RFC proposes that templates compiled via the `precompile` function
resolve unknown identifiers from the ambient JavaScript scope, rather than
performing dynamic resolution through the container.

To do this, `precompile` emits JavaScript code that contains references to
the same identifiers as those used in the template:

```js
import { precompile } from '@ember/template-compiler';

const wireFormat = precompile(`{{t "Hello!"}} <User @user={{this.currentUser}} />`);
/* => `{
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "scope": () => [t, User]
}`                ^---- template identifiers emitted as JavaScript references
*/
```

If a template references an identifier that is not in lexical
scope, a `ReferenceError` will be produced at runtime:

```js
import { precompileTemplate } from '@ember/template-compilation';

// Compiles successfully but the generated code will throw a ReferenceError when
// evaluated.
const wireFormat = precompile(`<MissingComponent />`);
/* => `{
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "scope": () => [MissingComponent]
}`                ^------ will produce ReferenceError
*/
```

Optionally, you can provide an array of strings specifying the ambient
identifiers in scope. If a `scope` array is provided and the template
contains an identifier that is not present in the list, that reference will
fall back to today's dynamic resolution lookup behavior:

```js
// While `t` comes from ambient JavaScript scope, `User` will be looked up
// through the container.
const wireFormat = precompile(`{{t "Hello!"}} <User @user={{this.currentUser}} />`, {
  mode: 2,
  scope: ['t']
}); /* => `{
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "scope": () => [t]
}`                ^----- only includes identifiers specified in `scope`
*/
```

```js
import { precompile } from '@ember/template-compilation';

const wireFormat = precompile(`<MissingComponent />`, {
  scope: ['BlogPost', 'OtherComponent']
})
/* => Throws an exception: "MissingComponent is not defined."   */
```

Addons that need to support existing templates that rely on container lookup
to discover components and helpers should provide the list of valid
identifiers via `scope`. Omitting this list causes all lookups to happen via
ambient scope rather than through the container.

As of this RFC, scope values resolved via ambient scope are limited to
component classes and template helpers (i.e. subclasses of `Helper` or
functions wrapped with `helper()` from `@ember/component/helper`). However,
support for additional kinds of scope values may be expanded in future RFCs.

### Compilation Modes

To support future evolution of template syntax and semantics, this RFC
defines a `mode` option that can specify different rules for template
compilation.

This RFC defines a single mode:

   Mode  | ID | Description
---------|--------------|------------
**Classic** | 1 | All deprecated and non-deprecated syntax supported. Mix of dynamic resolution and JavaScript scope supported.

Modes are specified as an integer ID to reduce the size of compiled output
and to allow additional modes to be added in the future. To specify the mode,
pass a `mode` option to `precompile`:

```js
const wireFormat = precompile(`{{t "Hello!"}} <User @user={{this.currentUser}} />`, {
  mode: 1,
  scope: ['t']
}); /* => `{
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "scope": () => [t]
}`
*/
```

If the mode is not specified, classic mode (mode `1`) is assumed. Future RFCs
may specify additional modes.

In strict mode, templates that references names that are not present in a supplied
`scope` array produce a compile-time error. In sloppy mode, references to names
that are not specified in the `scope` array will fall back to dynamic
resolution:

```js
// `t` comes from ambient scope, `User` will be looked up.
const wireFormat = precompile(`{{t "Hello!"}} <User @user={{this.currentUser}} />`, {
  mode: 2,
  scope: ['t']
}); /* => `{
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "scope": () => [t]
}`                ^----- only includes identifiers specified in `scope`
*/
```

## How we teach this

### Guide-level Documentation

This is a low-level API, and most Ember developers should never need to learn
or care about it. Only addon authors interested in providing alternate file
formats need to understand these APIs in depth.

As such, the examples and explanations given in this RFC should be enough for
experienced addon authors to get started. Of course, familiarity with
Broccoli, parsers like Babel, and creating Ember CLI addons is a
pre-requisite.

### API-Level Documentation

#### precompile

```js
import { precompile } from '@ember/template-compiler';
```

Compiles a string of Glimmer template source code into an intermediate "wire
format" and returns it as a string of JavaScript code. The exact value
returned from `precompile` is not specified, is not considered public API,
and is likely to change across minor versions. The string is guaranteed to be
a valid JavaScript expression. Other than that, you should treat the value
returned by `precompile` as completely opaque.

By default, components and helpers referenced in the passed template will be
looked up from the JavaScript scope where the wire format string is inserted.
To enable legacy container resolution behavior, a `scope` option can be
provided containing a list of identifiers that should be considered to come
from ambient JavaScript scope. Any identifiers in the template that are not
present in the `scope` list will fall back to dynamic container resolution at
runtime.

An optional `mode` option may also be provided. Currently, only one mode is
supported, which is the classic mode with ID of `1`. If no mode is specified,
the default mode of `1` is assumed. Future RFCs may specify additional modes.

```ts
function precompile(templateSource: string, options?: PrecompileOptions): string; 

interface PrecompileOptions {
  scope?: string[];
  mode?: PrecompileMode;
}

enum PrecompileMode {
  Classic = 1;
}
```

#### createTemplateFactory

```js
import { createTemplateFactory } from '@ember/template-factory';
```

Creates a wrapper object around the raw wire format data of a compiled
template. If the template had additional identifiers added to its ambient
scope when it was compiled with `precompile`, a `scope` function must be
provided when calling `createTemplateFactory` that returns an array of
reified values corresponding to each identifier.

```ts
function createTemplateFactory(wf: WireFormat): TemplateFactory;
```

#### setComponentTemplate

```js
import { setComponentTemplate } from '@ember/template-factory';
```

Associates a template factory with a component class. When a component is
invoked, a new instance of the class is created and the template associated
with it via `setComponentTemplate` is rendered.

```ts
function setComponentTemplate<T>(template: TemplateFactory, ComponentClass: T): T;
```

## Drawbacks

### Requires Addons

The most obvious drawback is that the APIs outlined here, once implemented,
stabilized, documented, and landed in a stable release, do not provide any
value to Ember users if no one build addons that take advantage of them.

On the other end of the spectrum, there's the risk that people build _too many_ addons
using these primitives, causing fragmentation and confusion in the Ember ecosystem.

There's also a risk that people find single-file component addons extremely
productive and they become popular among "pro users," but new Ember learners
aren't aware of them. Once these ideas are validated and shown to be an
improvement, it's important they are incorporated into the happy path so we
can climb the mountain together.

### Relies on Runtime Behavior

While this proposal tries to be strict about reducing the dynamism around how
component templates are specified, it still relies on having objects like
template factories available to imperatively annotate at runtime.

If and when we add support for Glimmer's AOT bytecode compilation mode, these
objects will no longer exist, and the strategy of having addons simply
compile the template portion of single-file components into an inline wire
format will no longer work.

While I don't consider this a blocker, it does mean that when the time comes,
existing SFC addons would likely need to be updated to support AOT
compilation, unless we can come up with a clever compatibility hack.

## Alternatives

### Providing Scope Explicitly

As currently proposed in this RFC, template identifiers are looked up from
ambient scope, with an optional fallback to dynamic resolution behavior.

An alternative considered was to specify scope values explicitly when
creating the template factory. However, this adds boilerplate that is
unneeded in the most common scenarios.

In favor of the extra boilerplate, addon authors can inject new bindings into
JavaScript scope to customize template resolution.

### Additional Resolver-Based APIs

One alternative to the API propsed here (where we explicitly attach a
template to a component class) would be to provide better hooks into the
resolver system so that addons can better simulate single-file components via
the existing dynamic resolution system.

I rejected this approach for a number of reasons. First, the current resolver
already has quite a bit of branching and dynamic behavior, often for
backwards compatibility. Adding additional functionality and logic to this system
did not seem like a great option.

Second, I believe the API proposed in this RFC is much simpler for addon
authors to understand and implement if they want to experiment with
single-file component file formats. More dynamic systems can also add more
layers of indirection that are difficult to understand and debug.

Third, and most importantly, the static nature of the API as proposed here
makes it much easier to understand at build time which components and
templates go together, and what the dependencies between components are. The
ability to quickly and accurately determine this dependency graph is
imperative to the success of initiatives like Embroider and is required to
unlock good tree-shaking and code-splitting.

## Unresolved questions

### Intermediate Wire Format

Given that the wire format is unstable between different versions of Ember,
it's important that all templates in an app (including those from addons) be
compiled together.

One of [Embroider's](https://github.com/embroider-build/embroider) goals is
ensuring that addons can compile their assets when the addon is published to
npm, rather than having compilation happen over and over again every time the
host Ember application is built.

We want an intermediate format that would allow addons to distribute
components authored with SFCs and template imports, but would defer final
template compilation to the wire format until the application was is
compiled.

## Changelog

This section captures substantive changes made during the RFC process and the
rationale behind the changes.

### 2019/2/25

#### Simpler Scope API

The details of how scope works have changed to better align with the planned
"strict mode" RFC for templates, and to reduce the boilerplate required in
most common scenarios.

Previously, setting a template's scope was a two-step process:

1. Specifying the identifiers during the `precompile` phase.
2. Specifying the associated values of the specifiers during the
   `createTemplateFactory` phase.

The API looked like this:

```js
import { precompile } from '@ember/template-compiler';
import { createTemplateFactory } from '@ember/template-factory';
import User from './OtherComponent';
import t from '../helpers/i18n/t';

// Provide array of string identifiers
const wireFormat = precompile(`{{t "Hello!"}} <User @user={{this.currentUser}} />`, {
  scope: ['User', 't']  
});

// Provide array of values
const template = createTemplateFactory(wireFormat, {
  scope: () => [User, t]
});
```

This API has been updated so that, by default, the expression returned from
`precompile` is assumed to be embedded in JavaScript code where the
identifiers used in the template are in lexical scope. This eliminates the
need to specify a `scope` array to either `precompile` or
`createTemplateFactory`.

The example above would instead be written like this:

```js
import { precompile } from '@ember/template-compiler';
import { createTemplateFactory } from '@ember/template-factory';
import User from './OtherComponent';
import t from '../helpers/i18n/t';

// Assume any ambiguous identifiers come from our lexical scope.
const wireFormat = precompile(`{{t "Hello!"}} <User @user={{this.currentUser}} />`);
const template = createTemplateFactory(wireFormat);
```

That means that the wire format can now contain JavaScript code, like in the
following example:

```js
import { precompileTemplate } from '@ember/template-compilation';
import User from './OtherComponent';
import t from '../helpers/i18n/t';

// Provide array of string identifiers
const wireFormat = precompile(`{{t "Hello!"}} <User @user={{this.currentUser}} />`);
/* => `{
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "scope": () => [t, User]
}` */
```

#### Mode Option

A new `mode` option has been added to `precompile` to allow for future
evolution of compiler rules.