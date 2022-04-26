---
Stage: Accepted
Start Date: 2022-04-16
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/813
---

# Layering and Desugaring for First-Class Component Templates

## Summary

This RFC propose to introduce a `template()` function as an substitution for
the `<template>` syntax extension defined in [RFC #779][rfc-779] using
standard JavaScript, for use in environments where the custom syntax extension
is not available, and also as a publishing format for addons on NPM:

[rfc-779]: ./0779-first-class-component-templates.md

```js
import { template } from "@ember/template-compilation";
import Hello from "my-app/components/hello";

// <template><Hello /></template> becomes...
export default template(`<Hello />`, () => ({ Hello }));

// export const Foo = <template><Hello /></template> becomes...
export const Foo = template(`<Hello />`, () => ({ Hello }));

// export class Bar {
//   <template><Hello /></template>
// }
export class Bar {
  static {
    template(`<Hello />`, () => ({ Hello }), this);
  }
}
```

Idiomatic usages of the `template()` function can be pre-compiled by a build
step if desired, but a runtime implementation can also be used via an opt-in
described below.

We also propose an alternative form that uses the `template()` function as a
tag function for JavaScript tagged template strings. However, this variant does
not have a runtime implementation and must be handled by a build step:

```js
import { template } from "@ember/template-compilation";
import Hello from "my-app/components/hello";

// <template><Hello /></template> becomes...
export default template`<Hello />`;

// export const Foo = <template><Hello /></template> becomes...
export const Foo = template`<Hello />`;

// export class Bar {
//   <template><Hello /></template>
// }
export class Bar {
  static {
    template`<Hello />`;
  }
}
```

In addition, we also propose a new `@ember/template-compilation/runtime`
module as an opt-in to enable runtime template compilation.

## Motivation

[RFC #779][rfc-779] introduced a first-class component syntax feature that has
numerous benefits, among which are:

1. Opt-in to strict mode ([RFC #496][rfc-496])
2. Eliminating the need for name-based runtime resolutions
3. Ability to use imported template constructs
4. Ability to interact with the surrounding JavaScript scope

[rfc-496]: ./0496-handlebars-strict-mode.md

However, one of the drawbacks is that it requires a custom syntax extension
(`<template>`) and a custom file format (`.gjs/.gts`), which requires a build
step and other custom tooling.

RFC #779 makes a good case for why this is necessary and preferable to the
alternatives. We stand by the rationales given in RFC #779, and this RFC does
not intend to revisit its recommendations. However, there remain cases where
this drawback is highly undesirable or simply not acceptable:

1. Until we have implemented good editor integration, the developer experience
   of using `.gjs`/`.gts` could be abysmal (without any syntax highlighting or
   completion at all, etc). Even after we ship good editor integration for the
   mainstream editors, this will likely remain the case for a subset of less
   used editors that we did not or could not prioritize supporting.

2. There may be editors and environments that are impossible for us to support,
   either because they don't have an extension system at all, or those systems
   do not expose the capabilities we need. For example, CodeSandbox, CodePen,
   TypeScript playground, etc.

3. When publishing packages to NPM, it is important and desirable to publish
   only standard `.js` files that does not require further processing by the
   consumer (see [v2 Addon Format](./0507-embroider-v2-package-format.md)).

4. There are use cases that prefer or require components that are directly
   runnable in the browser without a build step, such as bug report templates,
   runnable code samples, generating components dynamically at runtime, etc.

From here on, we will refer to users and use cases collectively as "standard
JavaScript environments". Under these circumstances, users would be unable to
adopt the `<template>` feature, which means they cannot take advantage of the
numerous benefits unlocked by the feature, such as the strict mode opt-in.

Also, if an addon chose to only provide template constructs (e.g. helpers and
components) exclusively via importable modules (as opposed to merging them into
"App Javascript"), then there will be no easy way to access these template
constructs from those environments.

It is not technically impossible to accomplish these same goals in standard
JavaScript environments. After all, the `<template>` feature is specified using
primitives that already exist. Users in standard JavaScript environments can
just use those primitives directly.

However, that is not an ideal outcome. `<template>` is a user-facing feature,
and is poised to (or at least well-positioned to) become the main way Ember
users read, write and reason about components in the next edition of Ember when
the feature is fully rolled out.

On the other hand, the primitives it is based on are very low-level, verbose,
error-prone and were more intended as a compilation target than an user-facing
authoring format. Their low-level nature and flexibility also makes it easy to
get the details wrong, such as forgetting to pass the `strictMode: true` flag
and or deviate from the normal lexical scope capturing rules. It would be
unfortunate if being restricted to the standard JavaScript environment also
means dropping down to a completely different programming model for components.

Currently, the addon [ember-template-imports][ember-template-imports] provides
an alternative that does work in regular `.js`/`.ts` files – the "hbs backtick"
format. However, it is not a good candidate for solving these problems for a
few reasons, some of which were the same reasons RFC #779 chose not to go that
route:

[ember-template-imports]: https://github.com/ember-template-imports/ember-template-imports

1. The feature was never specified in RFC #779 or any other RFC. It was added
   to the addon when its purpose was to be a sandbox for experimentation. With
   the acceptance of RFC #779, the experimentation phase is over and the addon
   should be transitioned into a compliant polyfill, which means removing the
   alternative `hbs` form.

2. The naming conflicts with the "official" `hbs` from `ember-cli-htmlbars`.
   This would have been fine if we chose to go that route in RFC #779 and have
   plans to evolve the official version to have the same feature. With RFC #779
   favoring the `<template>` approach, this is not going to happen and this
   conflict will now be quite confusing, especially since they actually do very
   different things (e.g. the "official" `hbs` does not return a component).

3. It uses a standard JavaScript syntax in a non-standard way, semantics-wise.
   It looks like an inert JavaScript string but has access to the lexical scope
   around it (without using the `${ ... }` syntax), which can be confusing. It
   also means that it is impossible to provide a runtime implementation using
   the same syntax that works in environments that does not permit a build step.

4. It uses `static template = ` to associate templates to classes. This implies
   runtime behavior that is actually not true (e.g. TypeScript will believe the
   `template` property exists on the class).

5. Values consumed by the template but is otherwise unused in the rest of the
   file may generate errors/warnings from linters or language servers.

To address these issues, this RFC propose that we introduce a new `template()`
function that servers as a middle ground between the `<template>` language
extension and the low-level primitives.

- It is designed to have the same semantics as the `<template>` feature (such
  as the strict-mode opt-in, returning a template-only component when not
  attached to a class), providing the same high-level programming model for
  authoring components in standard JavaScript environments.

- It requires manually supplying the lexical scope variable bindings (the
  "scope function").

- It uses a [static initializer block][static-block] to associate the template.

  [static-block]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Class_static_initialization_blocks

- While it is more cumbersome to use and have a degraded experience (e.g.
  possibly lacking hbs syntax highlighting) compared to first-class component
  templates, it is still designed to be ergonomic to use, to the extent
  possible in standard JavaScript environments.

- It is designed to be pre-processed at build time where possible, just like
  the `<template>` feature and today's "official" `hbs` tag. However, in cases
  where this is not possible, there will be a runtime implementation available
  provided the template compiler is also available at runtime.

- For environments where it is possible to configure the development and build
  tools to recognize the format, the tagged template literal form can be used,
  which automatically captures the lexical scope variable bindings. This
  directly replaces [ember-template-imports][ember-template-imports]'s "hbs
  backtick" format but brings its naming into alignment with the `<template>`
  feature.

  This form will produce an error at runtime if not pre-processed out
  as a correct runtime implementation is impossible.

  Despite the drawbacks mentioned above, this format may still be preferred as
  a transitional tool for early adopters while editor integrations are being
  worked on. It may also be useful for communicating with code snippets in
  platforms where syntax highlighting is not yet available for the custom
  `.gjs` format (such as Discord and GitHub today), but where it's desirable to
  still retain the highlighting for the JavaScript/TypeScript portions.

This RFC also _recommends_ that the addon spec (v2.1?) to be updated to
consider using `template()` (the function call version, not the tagged template
literal form) as the primary publishing format for component templates and
consider reducing and deprecating the need for the alternative mechanisms (such
as shipping `.hbs` modules). However, this is only a general recommendation,
and a future RFC or amendment is needed to explore that further.

That being said, such an update is only needed for the purpose of clarifying
the _recommended_ way of shipping templates in addons. Because `template()` is
just standard JavaScript, there is nothing stopping addons from adopting the
format as long as their target Ember versions support the feature, or that they
depend on a suitable polyfill.

## Detailed design

This RFC proposes to re-specify the `<template>` language extension into a
de-sugaring into `template()` calls:

1. Top-level declaration

   ```gjs
   import Foo from "somewhere";

   <template>
     <Foo />
   <template>
   ```

   becomes

   ```js
   import { template } from "@ember/template-compilation";
   import Foo from "somewhere";

   export default template(
     `
     <Foo />
   `,
     () => ({ Foo })
   );
   ```

2. Expression

   ```gjs
   import Foo from "somewhere";

   export const Bar = <template><Foo /><template>;
   ```

   becomes

   ```js
   import { template } from "@ember/template-compilation";
   import Foo from "somewhere";

   export const Bar = template(`<Foo />`, () => ({ Foo }));
   ```

3. Class

   ```gjs
   import Foo from "somewhere";

   export default class Bar {
     <template><Foo /><template>
   }
   ```

   becomes

   ```js
   import { template } from "@ember/template-compilation";
   import Foo from "somewhere";

   export default class Bar {
     static {
       template(`<Foo />`, () => ({ Foo }), this);
     }
   }
   ```

   When used in this form, the function returns/passes through the target
   supplied in the third argument (the component class).

   Note that the use of a static initializer block here is important for
   capturing access to private fields. This will be discussed in a subsequent
   section.

In these small snippets, the scope bindings may look very verbose compared to
the size of the templates, but in real-world templates, the ratio will improve.

This de-sugaring specified in this section is intended to **subsume** the parts
of RFC #779 which references its compilation output, and where it specified the
feature in terms of the low-level primitives such as `precompileTemplate()`. If
this RFC is accepted, the `<template>` feature should only be understood in
terms of `template()`, which will be a stable and public API that stands on its
own.

Any build-time optimization that pre-compiles the `template()` calls at build
time are required to ensure their output to have equivalent semantics as the
runtime `template()` calls, assuming the same set of AST plugins are applied.
Their exact compilation output should be considered private implementation
details which may change over time and should not be relied upon.

Once this RFC is accepted, any future changes (including deprecations) to the
primitive APIs referenced in RFC #779 should have no direct bearing on the
`<template>` or `template()` features.

### Tagged Template Literals

Alternatively, the `template()` function can also be used as a tag function for
a JavaScript tagged template string. This alternate form automatically captures
the lexical scope variable bindings. However, this form does not have a runtime
implementation and must be processed by a build step.

1. Top-level declaration

   ```gjs
   import Foo from "somewhere";

   <template>
     <Foo />
   <template>
   ```

   becomes

   ```js
   import { template } from "@ember/template-compilation";
   import Foo from "somewhere";

   export default template`<Foo />`;
   ```

2. Expression

   ```gjs
   import Foo from "somewhere";

   export const Bar = template`<Foo />`;
   ```

   becomes

   ```js
   import { template } from "@ember/template-compilation";
   import Foo from "somewhere";

   export const Bar = template`<Foo />`;
   ```

3. Class

   ```gjs
   import Foo from "somewhere";

   export default class Bar {
     <template><Foo /><template>
   }
   ```

   becomes

   ```js
   import { template } from "@ember/template-compilation";
   import Foo from "somewhere";

   export default class Bar {
     static {
       template`<Foo />`;
     }
   }
   ```

   Again, the use of a static initializer block is important for capturing
   access to private fields.

   In this form, the tagged template string literal _must_ be the **only**
   statement in the static initializer block. However, the JavaScript standard
   permits multiple static initializer blocks in the same class body, so any
   other initialization logic can be placed in separate blocks as needed. It is
   an error for a class body to contain multiple templates associated this way,
   just as it is for a class body to contain multiple `<template>` blocks.

### Access to Private Fields

The `<template>` feature proposed by RFC #779 explicitly chose not to provide
access to private fields when a `<template>` is associated with a class, citing
limitations to the compilation output, specifically that the compiled template,
as proposed, would be placed outside of the class body, rendering it impossible
to access any private fields. RFC #779 acknowledges that is a gap that should
be addressed in a future RFC.

With the advancement of [static initialization blocks][static-block], it is
now possible to place the associated template directly inside the class body.
This RFC takes advantage of this and propose an adornment to allow access to
private fields inside `<template>` tags and the proposed `template()`:

```gjs
import { tracked } from "@glimmer/tracking";

export default class Foo {
  @tracked #value;

  <template>
    My value is {{this.#value}}.
    The other value is {{@other.#value}}.
  </template>
}
```

```js
import { template } from "@ember/template-compilation";
import { tracked } from "@glimmer/tracking";

export default class Foo {
  @tracked #value;

  static {
    template`
      My value is {{this.#value}}.
      The other value is {{@other.#value}}.
    `;
  }
}
```

```js
import { template } from "@ember/template-compilation";
import { tracked } from "@glimmer/tracking";

export default class Foo {
  @tracked #value;

  static {
    template(`
      My value is {{this.#value}}.
      The other value is {{@other.#value}}.
    `, () => ({ '#value': (_) => _.#value }, this);
  }
}
```

As shown in this last example, it requires a small amendment to the "scope
function". The scope function was first introduced in RFC #496, and then
subsequently modified slightly from returning an array to an object literal.

The purpose of the scope function is to provide the rendering layer with the
needed lexical scope variable bindings that are referenced in the template,
in the form of key-value pairs in the returned object literal, where the key
is the identifier (variable) as referenced in the template, and the value is
the value referenced by the identifier.

To support private field access, we propose to extend this format to allow
keys that starts with a leading `#` character. For each unique private field
reference in the template (any `.#` access), a corresponding entry must be
added to the object literal where the key matches the name of the private
field in string form. This is not a legal JavaScript identifier thus it does
not generate a conflict with the current format, and will require quoting. The
value of the entry is expected to be a JavaScript "accessor function" that
takes a single argument – an object instance of the same class – and returns
the current value of the private field at the time of the call.

This will require an update to the glimmer-vm rendering engine to support this
format. Internally, it will compile the template in such a way that any access
to a private field in the template will go through the corresponding accessor
function. Since the accessor functions are embedded inside the class body, they
will be permitted to access the private fields declared for the class. It is
therefore crucial that the `template()` call is embedded inside a static
initializer block in the class body.

With this change, we can fully support private fields in templates. As shown in
the examples, this applies to both direct access (`this.#field`) as well as
indirect access (`some.nested.object.of.the.same.kind.#field`), as long as it
is of the same instance type, fully matching the JavaScript semantics.

### Build-time Transformations

Typically, it is desirable to for Ember apps to precompile all templates at
build time. By default, Ember does not include a template compiler in the
bundle as it adds significant amount of code but most apps do not have a use
for it.

The `template()` function proposed in this RFC is designed so that idiomatic
usages (defined in the next section) can be processed by a built step, which
is enabled by default.

Non-idiomatic usages are not guaranteed to be transformable at build time. If a
transformation cannot be applied, the call will be left as-is for runtime
processing. Note that since the runtime template compiler is not available by
default (see below for how to opt-in), this default behavior will result in a
runtime error.

Because the rules for determining idiomatic usages are purely syntactical, they
can also be checked by a linter. We propose to include a enabled-by-default
lint rule to warn against accidental non-idiomatic usages.

The purpose of specifying an idiomatic subset of the possible syntaxes and
compositions is to reflect that this feature is intended to be an equivalent
of the `<template>` feature in environments where the custom extensions are not
viable. Therefore, developers should try to stick to the same set of semantics
and affordances provided by `<template>`. It also defines the common subset of
features that tools like codemods and linters must be able to handle.

Deviating outside of this subset of idiomatic uses may result in degraded
performance (due to requiring runtime template compilations) and development
experience (due to tools not able to make static analysis on the templates), so
developers should be very aware of these tradeoffs and should be a deliberate
choice.

That being said, we acknowledge that there are legitimate use cases for wanting
the extra flexibility or needing runtime behavior. By including a runtime
implementation of the feature (again, see below), we get handling for all the
remaining edge cases "for free".

#### Idiomatic Usages

In either form, the `template()` function must be imported directly from the
`@ember/template-compilation` module for the invocation to be considered
idiomatic. Renaming the import binding is permitted. The import binding must
be used directly in the invocations/call-sites. For example, if the imported
value is stored into another constant storing the imported into another
variable, then any indirect invocations are not considered an idiomatic use.

For the function call form of the `template()` function, idiomatic usages are
defined as:

1. Unless otherwise specified:

   1. All string literals inside the call (arguments, property keys, etc) must:
      1. Be quoted with single quotes, or
      2. Be quoted with double quotes, or
      3. Be quoted with backticks but without a tag and without interpolations.
   2. All object literals ("POJOs") inside the call must:
      1. Not contain duplicate property names, and
      2. Not contain computed property names, even if the computed name is just
         a string literal, and
      3. Not contain any getters, setters or methods, and
      4. Not use the spread operator.
   3. All function expressions must:
      1. Be specified with either the `function` keyword or arrows, and
      2. Be anonymous, and
      3. Not have any modifiers (e.g. `async`, `function*`, etc), and
      4. Have exactly one statement in the function body, which must be a
         `return` statement. For arrow function expressions, this implies the
         single-expression implicit return shorthand syntax can be used, which
         is encouraged but optional.

2. The first argument to the call must be a string literal containing the
   template source.

3. The second argument to the call is optional. If provided, it must be a
   function expression that:

   1. Has no parameters, and
   2. Returns an object literal where every name-value pair must be in the form
      of:
      1. The property name must be a valid JavaScript identifier and its
         value must be a reference to the variable with the same name. This
         implies that the JavaScript shorthand property syntax can be used,
         which is encouraged but optional. Alternatively,
      2. When the `template()` function is called from within a `static` block
         inside a class body, the property name can also be a valid JavaScript
         private field name (i.e. a valid JavaScript identifier prefixed by the
         `#` character), which must be quoted. Its value must be a function
         expression that:
         1. Has exactly one parameter, and
         2. Returns the current value of the corresponding private field (with
            the same name as the object key) on the object passed in the
            parameter.

4. When, and only when, the `template()` function is called from within a
   `static` block inside a class body, then a third argument must be supplied
   to the call, and this argument must be the `this` keyword.

5. No additional arguments can be passed.

For the tagged template string literal form, idiomatic usages are defined as:

1. The string literal must not contain any interpolations.
2. For class association, the tagged template string must be placed inside a
   `static` block within the class body, where it must be the only statement in
   the block.

Some examples of idiomatic usages:

```js
template("Hello world!");
template(`Hello world!`);

template("Hello world!", () => ({}));
template("Hello world!", () => {
  return {};
});
template("Hello world!", function () {
  return {};
});

template("<Hello />", () => ({ Hello }));
template("<Hello />", () => {
  return { Hello };
});
template("<Hello />", function () {
  return { Hello };
});

// Quoting the property names are also allowed
template("<Hello />", () => ({ Hello: Hello }));
template("<Hello />", () => {
  return { Hello: Hello };
});
template("<Hello />", function () {
  return { Hello: Hello };
});

class Foo {
  static {
    template("<Hello />", () => ({ Hello }), this);
  }
}

class Bar {}

template("<Hello />", () => ({ Hello }), Bar);
```

Some examples of non-idiomatic or incorrect usages:

```js
template();

let t = template;
t("Hello world!");

let source = "Hello world!";
template(source);

template("Hello" + " " + "world!");
template(`Hello ${audience}!`);
template(stripIndent`
  Hello
`);

template("Hello world!", () => {});

const EMPTY_OBJECT = {};
template("Hello world!", () => EMPTY_OBJECT);

template("Hello world!", function scope() {
  return {};
});

template("Hello world!", /* this is a comment */ () => {});

template("<Hello />", () => ({ ...scope }));
template("<Hello />", () => ({ ["Hello"]: Hello }));
template("<Hello />", () => {
  let scope = {};
  scope["Hello"] = Hello;
  return scope;
});

template("<Hello />", () => ({ Hello }), class Foo {});

class Bar {
  static template = template("<Hello />", () => ({ Hello }));
}

class Baz {
  static {
    debugger;
    template`<Hello />`;
  }
}

class Bat {
  static {
    let template = template`<Hello />`;
  }
}
```

As pointed out above, the philosophy around these restrictions is to maintain
parity between the possible semantics with the `<template>` and restricting the
syntactic variations to make it feasible for tools to perform the static
analysis reliably.

On the other hand, at least in some of the use cases we are targeting, these
would be hand-written in hand-maintained source code files (as opposed to only
being a compilation target). Therefore, it is also an important balancing act
to not add gratuitous restrictions that prescribe a "canonical syntax" that is
easy to violate by accident. For example, while would could mandate, say, that
the scope function must be specified using arrow functions, it could be easily
forgotten and get missed in code-reviews, which could subtly cause the template
to fallback to runtime compilation. Formatting tools like prettier may also
automatically rewrite the code between different equivalent styles.

If the idiomatic usage rules are followed, a build-time processor will be able
to provide warnings or errors for common mistakes, such as syntax errors within
the Handlebars source code, missing or unused scope variables, etc. It may also
be possible to provide some limited editor integration, though as pointed out
by [RFC #779][rfc-779], there are some inherit challenges with this approach.

### Runtime Compilation

Ember has always had the ability to compile template at runtime. However, since
this incur significant costs and most apps do not benefit from it, this feature
is disabled by default and requires and explicit opt-in to include the runtime
template compiler into the app's bundle. The mechanism of the opt-in will be
discussed later in this section.

If the template compiler is available at runtime, then it will be possible to
use the `template()` function at runtime as well. However, as mentioned above,
this does not extend to the "tagged template string" format. Attempting to use
the `template()` function as a tag at runtime will result in a helpful error in
debug builds and undefined behavior on production builds.

If the template compiler is not available at runtime, then attempts to call the
`template()` function at runtime will result in a helpful error in debug builds
and undefined behavior on production builds. On production builds, it is not
guaranteed that the `template` named export may be undefined (i.e. it may not
even be a function), but developers must not rely on this to "feature-detect"
the availability of runtime compilation.

Notably, even when template compilation is available at runtime, the result of
the compilations may be subtly different. This is because that applications may
have custom glimmer/handlebars AST plugins in their build, and these plugins
will not be available at runtime.

This may change in the future, but it has always been true for runtime template
compilation and will likely remain to be the case for the foreseeable future,
as these plugins are often authored or packaged in ways that are inherently not
browser-friendly. This usually only used for optional syntax extensions such as
"inline {{let}}" so in practice it is not an issue. However, some polyfills for
template features also uses the AST plugin API and those polyfilled features
will not be available to templates compiled at runtime.

#### Opting In

Today, applications can opt-in to bundling the runtime compiler with something
along these lines in their `ember-cli-build.js`:

```js
app.import("node_modules/ember-source/dist/ember-template-compiler.js");
```

The exact details depend on the version of `ember-source` and is not very well
documented. It also does not align with the direction we are headed, where we
encourage using the modules system and imports to express dependencies. This
configuration is also not friendly to tree-shaking/code-splitting, as it forces
the template compiler to be included into the main/initial bundle, when it may
only be needed in one of the infrequently used routes.

This RFC takes the opportunity to improve on this by proposing a new module –
`@ember/template-compilation/runtime`. When this module is imported, either as
a "side-effect" import statement, or if its exported values are imported, then
it serves as an opt-in for enabling runtime template compilation.

For the time being, we propose that this module should have a single named
export – `template`, which is a re-export of the `template()` function we
described above. It has the same semantics as `template()` from the main module
except it is always invalid to use this imported function as a tag, so linters
and TypeScript can trivial flag this invalid usage.

The purpose of the re-exported `template()` from the runtime module is to
signal to both human readers and build tools that the template compilation is
deliberately deferred to runtime. These usages must not be processed at build
time and linters must not warn about non-idiomatic usages.

Typically, developers should favor importing `template()` from the main module
so that their code can be agonistic against where the compilation is happening.
However, since the intent is that build-time compilation should be _possible_,
they should also take care to ensure they stay within the set of idiomatic set
of usages to guarantee that is the case, and this re-export from the runtime
module offers a way to signal that the deviations are deliberate.

This RFC does not propose to add re-exports for other features found in the
`@ember/template-compilation` package. Conceptually, it makes sense to provide
the same ex-export for at least the `compile()` function. However, as we move
to a world where components are used everywhere and the need for "standalone
templates" dwindles, it is unclear that the `compile()` function is needed in
the long run. In any case, if that turns out to be necessary, a future RFC can
propose to add support for that and the precedence set forth in this RFC should
make that a straightforward process.

### TypeScript

We don't expect this feature to pose any significant challenges in terms of
TypeScript support, other than having a few overloaded signatures/purpose on
the same `template()` function may make things a little cumbersome, but not
prohibitively so. The signature of the function can be described as such:

```ts
// These types are defined in other RFCs
interface Signature {
  /* ... */
}
interface Component<S extends Signature> {
  /* ... */
}

type Scope = () => Record<string, unknown>;

function template<S extends Signature>(source: string): Component<S>;
function template<S extends Signature>(
  source: string,
  scope: Scope
): Component<S>;
function template<S extends Signature, C extends Component<S>>(
  source: string,
  scope: Scope,
  target: C
): C;

// For the build-time tagged template literal extension. By not specifying
// additional arguments here, TypeScript actually enforces that the tagged
// string literal cannot have any interpolation expressions.
function template<S extends Signature>(
  source: TemplateStringsArray
): Component<S>;
```

One nice thing about this design is that there is a natural place for the type
(generic) argument for the template's signature, negating the need for the
special `<template[Signature]>` syntax.

## How we teach this

The teaching recommendations in [RFC #779][rfc-779] shall remain in full force.
We continue to recommend teaching materials to be focused around `<template>`
and `.gjs`. The `template()` feature is recommended only in standard JavaScript
environments where adoption of `<template>` and `.gjs` is not feasible.

Because `template()` is a standard JavaScript API that has a "real" import
location, there is a natural place to document and describe its behavior in the
API docs. This should be the primary reference of the feature, and should also
where we document the idiomatic subset as well.

In the section of the guides where we teach `<template>`, there should also be
a small subsection discussing using `template()` as an alternative, including
what the appropriate use cases and its potential drawbacks. There may also be
value in teaching `<template>` in terms of its desugaring into `template()` in
the guides. Some developers may find it easier to appreciate what `<template>`
is or does and its benefits by comparing it to the equivalent standard
JavaScript desugaring.

## Drawbacks

Exposing the ability to manually pass a scope function can expose flexibility
that we don't intend on providing and opening it up to misuse. For example, the
closure could do more than just simply capturing the surrounding lexical scope,
impart side-effects, observe the timing of the call, etc.

However, this is not a new problem in a sense, as the primitives already exists
and the current status quo is that users who cannot use `<template>` must use
those primitives directly. With good linting and the ability to precompile and
analyze templates statically, it provides good incentives for developers to
stick with the idiomatic subset, which is carefully defined to have the same
semantics and power as `<template>`.

In any case, a lot of these "dangerous" patterns are already possible outside
of the scope function, especially after the [default helper manager][rfc-756]
making it easy to define inline helpers.

[rfc-756]: ./0756-helper-default-manager.md

With the ability to fallback to runtime compilation, the existence of these
edge cases do not create a lot of additional implementation complexity.

## Alternatives

### Class Decorators

We could use class decorators to associate templates to classes instead of the
static initializer block approach. The biggest drawback is that we would not be
able to access private fields with this design. In addition, in practice it
feels pretty awkward to use due to the length of the source text and the amount
of arguments needed, even for trivial examples. It also doesn't help this case
that stage 3 decorators require that class decorators come after the `export`
and `default` keywords:

```js
export @template(`<Hello />`, () => ({ Hello }) class Bar {
}
```

### Reserving Space For "Attributes"

It seems likely that the `<template>` tag will eventually gain some kind of
"attributes" syntax for configuring things like whitespace handling (see the
next section).

One option is to "reserve space" for that now, but it can also be done later by
optionally allowing the current "scope closure" argument to be a options object
where the scope closure would be passed as `{ scope: () => ({ ... }) }`.

## Unresolved questions

### Whitespace Handling

One thing that [RFC #779][rfc-779] did _not_ specify is that how whitespace
should be handled. The default conclusion is probably that the whitespaces
should be preserved exactly the way they appear in the `.gjs` source text.
While this technically works, it is a conclusion that pleases and benefits no
one, and will likely be revised/refined during the "spec work" process before
the feature is shipped.

On the one hand, there is the argument that whitespaces are typically not
significant in almost all HTML contexts templates. Shipping all these useless
whitespace just to have the browser fold them away when rendered into the DOM
is a not insignificant amount of wasted space in app bundles today. It also
just creates unnecessary work for the browser and the rendering layer.

On the other hand, there is the argument that whitespaces _can_ be significant.
Sometimes, this can be observed from uses of the `<pre>` element, but really,
the whitespace handling for any element (including `<pre>`) can be changed
using the CSS `white-space` property, so it is not necessarily safe to assume
that we can fold/collapse whitespaces in templates just because that's the
default behavior of the browser.

However, even with that in mind, the default conclusion for the whitespace
handling in `<template>` tags is _still_ not useful, _especially_ when the
author is trying to preserve significant whitespace for something like code
samples. This is because `<template>` tags often appears in JavaScript
contexts that already has leading indentation (such as inside a class body), so
in practice, if we preserve the whitespace exactly as it appears inside the
`<template>` tag, it still would not do what the author was trying to
accomplish.

It seems likely that we would need to define some default (perhaps globally
configurable) whitespace handling rules, along with some mechanisms for local
overrides. Perhaps something like `<template white-space="pre">`.

Once we decide that for `<template>`, then we have to make the additional
choice of what to do about `template()`. Because they are the same feature,
they face the same challenges. Again, the default course of action would be to
do nothing/preserve the whitespaces exactly, but it's a solution that serves no
one for the same reasons.

It is the same challenge that multi-line backtick string literals faces in
normal JavaScript. Perhaps it would have been ideal if the language specified
backticks to have `stripIndent` semantics, but that ship has sailed now.

In the build time version of `template()`, we could certainly apply the same
whitespace handling semantics as we have the full source text available to us
for the analysis. Depends on what those semantics are, we may or may not be
able to do the same in the runtime version, and it may be a divergence that we
have to accept.

Of course, in those cases, since the compilation runs at runtime anyway,
developers can apply arbitrary whitespace normalization on the source text
themselves before passing them into the `template()` function, such as using
the popular [strip-indent][strip-indent] NPM package, so it does not create any
functional limitations per-se. The main issue here is whether we end up with
different semantics in this area between the build-time and runtime versions,
and whether it is acceptable to require the few developers needing runtime
compilation to know that the difference exists. Then again, because of the lack
of AST plugins at runtime, this may not be the only or even the most notable
difference in a given application anyway.

[strip-indent]: https://www.npmjs.com/package/strip-indent

In any case, we don't necessarily have to solve this problem as part of this
RFC, as we would still have to do the same amendment on `<template>`, with that
conclusion being the primary driver for the decision in `template()`. However,
it is import to come to some conclusion before either of these features is
officially shipped as it would be hard to change after.
