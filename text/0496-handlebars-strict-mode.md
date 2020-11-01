---
Start Date: 2019-02-14
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/496
Tracking: (leave this empty)

---

# Handlebars Strict Mode

## Summary

In this RFC, we propose a set of changes to Ember's variant of Handlebars that
are aimed at codifying best practices, improving clarity and simplifying the
language. Together, these changes are bundled into a "strict mode" that Ember
developers can opt-into. In contrast, the non-strict mode (i.e. what developers
are using today) will be referred to as "non-strict mode" in this RFC.

This RFC aims to introduce and define the semantics of the Handlebars strict
mode, as well as the low-level primitive APIs to enable it. However, it does
not introduce any new user-facing syntax or conveniences for opting into this
mode.

The intention is to unlock experimentation of features such as template
imports and single-file components, but those features will require further
design and iterations before they can be proposed and recommended to Ember
users.

## Motivation

Ember has been using Handlebars since it was released 7 years ago (!). Over
time, we have evolved, adapted and in some case repurposed the Handlebars
language significantly (remember "context-shifting" `{{#each}}`?). This RFC
proposes to provide a "strict mode" opt-in to remedy some of Handlebars' design
decisions that we have come to regret over the years, or are otherwise not a
good fit for Ember. We believe this will make the Handlbars language easier to
learn, understand and implement, as well as enable better tooling to support
common development workflows for Ember developers.

We propose the following changes:

1. No implicit globals
2. No implicit `this` fallback
3. No implicit invocation of argument-less helpers
4. No dynamic resolution
5. No evals (no partials)

### 1. No implicit globals

Today, Ember implicitly introduces a set of implicit globals into a template's
scope, such as built-in helpers, components, modifiers. Apps and addons also
have the ability to introduce additional implicit globals by placing files into
the `app` folder or broccoli tree. It is also possible to further influence
this behavior by using the intimate resolver API (such as the alternative
"pods" layout).

This adds a fair amount of dynamism, ambiguity and confusion when reading
templates. When an identifier is encountered, it's not always clear where this
value comes from or what kind of value it may be. This problem is especially
acute for the "ambigious content" position, i.e. `<div>{{foo-bar}}</div>`,
which could be a local variable `foo-bar`, a global component or a helper named
`foo-bar` provided by the app or an addon, or `{{this.foo-bar}}` (see the next
section). This problem is even worse when there is a custom resolver involved,
as the resolver may return a component or helper not found in the "expected"
location at runtime.

Not only is this confusing for the human reader, it also makes it difficult for
the Glimmer VM implementation as well as other ecosystem tooling. For example,
if the developer made a typo, `{{food-bar}}`, it would be impossible to issue
a static error (build time error, inline error in IDEs) because the value may
be resolvable at runtime. It is also difficult, and in some cases impossible,
to implement IDE features such as "Jump to definition" without running code.

[RFC #432](https://github.com/emberjs/rfcs/blob/master/text/0432-contextual-helpers.md#relationship-with-globals)
described some additional issues with the current implicit globals semantics.

We propose to remove support for implicit globals in strict mode. All values
must be explicitly brought into scope, either through block params or defined
in the "ambient scope" (see [Detailed design](#detailed-design) section).

### 2. No implicit `this` fallback

Today, when Ember sees a path like `{{foo}}` in the template, after exhausting
the possibilities of implicit globals, it falls back to `{{this.foo}}`. This
adds to the same confusion outlined above. More details about the motivation
can be found in the accepted [RFC #308](https://github.com/emberjs/rfcs/pull/308).

We propose to remove support for implicit `this` fallback in strict mode. The
explicit form `{{this.foo}}` must be used to refer to instance state, otherwise
it will trigger the same errors mentioned in the previous section.

It is worth mentioning that [RFC #432](https://github.com/emberjs/rfcs/pull/432)
laid out a transition path towards a world where "everything is a value".
Between this and the previous restriction, we essentially have completed that
transition. To recap, here is a list of the
[outstanding issues](https://github.com/emberjs/rfcs/blob/master/text/0432-contextual-helpers.md#relationship-with-globals):

1. Not possible to reference globals outside of invocation positions
2. Invocation of global helpers in angle bracket named arguments positions
3. Naming collisions between global components, helpers and element modifiers

All of these problems are all related to implicit globals and/or implicit
`this` fallback. Since neither of these features are supported in strict mode,
they are no longer a concern for us.

### 3. No implicit invocation of argument-less helpers

In the contextual helpers RFC, we discussed [an issue](https://github.com/emberjs/rfcs/blob/master/text/0432-contextual-helpers.md#relationship-with-globals)
regarding the invocation of arguments-less helpers in invocation argument
positions.

In today's semantics, if there is a global helper named `pi`, the following
template will result it the helper being invoked, and the result of the helper
invocation will be passed into the component.

```hbs
<MyComponent @value={{pi}} />
```

This is not desirable behavior in the value-based semantics proposed in that
RFC, because it makes it impossible to pass helpers around as values, just as
it is possible to pass around contextual components today.

The contextaul helper RFC proposed to deprecate this behavior and require
mandatory parentheses to invoke the helper.


```hbs
{{!-- passing the helper pi to the component --}}
<MyComponent @value={{pi}} />

{{!-- passing the result of invoking the helper pi to the component --}}
<MyComponent @value={{(pi)}} />
```

In strict mode, the current, soon-to-be-deprecated behavior will be removed,
and the parentheses syntax will be mandatory.

Note that this only affects arguments-less helpers, which are exceedingly rare,
as most helpers perform self-contained computations based on the provided
arguments. It also only affect argument positions. In content and attribute
positions, the intent is clear as it does not make sense to "pass a helper into
the DOM". The parentheses-less form will continue to work in those positions,
although the explicit parentheses are also permitted.

### 4. No dynamic resolution

Today, Ember supports passing strings to the `component` helper (as well as the
`helper` and `modifier` helpers proposed in [RFC #432](https://github.com/emberjs/rfcs/pull/432)).
This can either be passed as a literal `{{component "foo-bar"}}` or passed as a
runtime value `{{component this.someString}}`. In either case, Ember will
attempt to _resolve_ the string as a component.

It shares some of the same problems with implicit globals (where did this come
from?), but the dynamic form makes the problem more acute, as it is difficult
or impossible to tell which components a given template is dependent on. As
usual, if it is difficult for the human reader, the same is true for tools as
well. Specifically, this is hostile to "tree shaking" and other forms of static
(build time) dependency-graph analysis, since the dynamic form of the component
helper can invoke _any_ component available to the app.

We propose to remove support for these forms of dynamic resolutions in strict
mode. Specifically, passing a string to the `component` helper (as well as the
`helper` and `modifier` helpers), whether as a literal or at runtime, will
result in an error.

In practice, it is almost always the case that these dynamic resolutions are
switching between a small and bounded number of known components. For this
purpose, they can be replaced by patterns similar to this.

```js
import Component from '@glimmer/component';
import { First, Second, Third } from './contextual-components';

class ProviderComponent extends Component {
  get selectedComponent() {
    switch(this.args.selection) {
      case "first":
        return First;
      case "second":
        return Second;
      case "third":
        return Third;
    }
  }
}
```

```hbs
{{yield this.selectedComponent}}
```

This will make it clear to the human reader and enable tools to perform
optimizations, such as tree-shaking, by following the explict dependency graph.

### 5. No evals (no partials)

Ember currently supports the `partial` feature. It takes a template name, which
can either be passed as a literal `{{partial "foo-bar"}}` or passed as a
runtime value `{{partial this.someString}}`. In either case, Ember will resolve
the template with the given name (with a prefix dash, like `-foo-bar.hbs`) and
render its content _as if_ they were copy and pasted into the same position.

In either case, the rendered partials have full access to anything that is "in
scope" in the original template. This includes any local variables, instance
variables (via implicit `this` fallback or explicitly), implicit globals, named
arguments, blocks, etc.

This feature has all of the same problems as above, but worse. In addition to
the usual sources (globals, `this` fallback etc), each variable found in a
partial template could also be coming from the "outer scope" from the caller
template. Conversely, on the caller side, "unused" variables may not be safe
to refactor away, because they may be consumed in a nested partial template.

Not only do these make it difficult for humans to follow, the same is true for
tools as well. For example, linters cannot provide accurate undefined/unused
variables warning. Whenever the Glimmer VM encounter partials, it has to emit
a large amount of extra metadata just so they can be "wired up" correctly at
runtime.

This feature has already been deprecated, and we propose to remove support for
partials completely in strict mode. Until the feature has been removed from
Ember itself, invoking the `{{partial ...}}` keyword in strict mode will be a
static (build time) error.

The use case of extracting pieces of a template into smaller chunks can be
replaced by [template-only components](https://github.com/emberjs/rfcs/pull/278).
While this requires any variables to be passed explicitly as arguments, it also
removes the ambiguity and confusions.

It should also be mentioned that the `{{debugger}}` keyword also falls into the
category of "eval" in today's implementation, since it will be able to access
any variable available to the current scope, including `this` fallback and when
nested inside a partial template. However, with the other changes proposed in
this RFC, we will be able to statically determine which variables the debugger
will have access to. Therefore we would still be able to support debugger usage
in strict mode without a very high performance penalty.

## Detailed design

This RFC aims to introduce and define the semantics of the Handlebars strict
mode, as well as the low-level primitive APIs to enable it. However, it does
not introduce any new user-facing syntax or conveniences for opting into this
mode.

The intention is to unlock experimentation of features such as template
imports and single-file components, but those features will require further
design and iterations before they can be proposed and recommended to Ember
users.

During this phase of experimentation, these low-level APIs will also enable
other build tools (such as [ember-cli-htmlbars](https://github.com/ember-cli/ember-cli-htmlbars))
to provide their own API for users to opt-in. In addition, once this RFC is
approved and implemented in Ember, these low-level APIs will have the same
semver guarentees as any other APIs in Ember, allow these experimentations to
be built on stable grounds.

### Low-level APIs

We propose to add an option to Ember's template compiler to enable strict mode
compilation.

There are three primitive APIs involved in compiling Ember templates, `precompile`,
`template` and `compile`.

The `precompile` function (a.k.a. [`precompileTemplate`](https://github.com/ember-cli/ember-rfc176-data))
is responsible for taking a template string, running AST plugins, checking for
errors and returning the "wire format" representation of the template. The
exact details of this "wire format" is unspecified and changes from time to
time across minor Ember versions. The only guarantee is that it returns a
string whose content is a valid JavaScript expression.

For example:

```js
import { precompileTemplate } from '@ember/template-compilation';

precompileTemplate('Hello, {{name}}!', {
  moduleName: 'hello.hbs'
}); /* => `{
  "id": "AoL2bkKU",
  "block": "{\"statements\":[\"...\"]}",
  "meta": {"moduleName":"hello.hbs"}
}` */
```

Again, the exact wire format changes from time to time, but the key is that the
content is valid JavaScript. This allows build tools to take this output and
insert it into any context where JavaScript expressions are allowed.

At runtime, the "wire format" can be "rehydrated" into something consumable by
Ember via the `template` function (a.k.a. [`createTemplateFactory`](https://github.com/ember-cli/ember-rfc176-data)).

Build tools typically compile templates into JavaScript modules by combining
these two pieces. In our example, the `hello.hbs` template is typically
compiled into a module similar to this:

```js
import { createTemplateFactory } from '@ember/template-factory';

export default createTemplateFactory({
  "id": "AoL2bkKU",
  "block": "{\"statements\":[\"...\"]}",
  "meta": { "moduleName": "hello.hbs" }
});
```

Finally, the `compile` function (a.k.a. [`compileTemplate`](https://github.com/ember-cli/ember-rfc176-data))
is a convenience helper that simply combines the two steps by taking a raw
template string and returning a ready-to-be-consumed template object (the
output of `createTemplateFactory`), instead of the wire format. This is
mostly used for compiling templates at runtime, which is pretty rare.

We propose to introduce a new `strict` option to the `precompile` and `compile`
functions to enable strict mode compilation:

```js
import { precompileTemplate } from '@ember/template-compilation';

precompileTemplate('Hello, {{name}}!', {
  moduleName: 'hello.hbs',
  strict: true
});
```

### The ambient scope

Since there are no implicit globals in strict mode, there has to be an
alternative mechanism to introduce helpers and components into scope.

Whenever the strict mode compiler encounters an undefined reference, i.e. an
identifier that is not a currently in-scope local variable (block param), the
default behavior is to assume that these are references to variables in the
_ambient scope_. That is, the compiler will emit JavaScript code that contains
JavaScript references to these variables.

For example, consider the following template:

```hbs
{{#let this.session.currentUser as |user|}}
  <BlogPost @title={{titleize @model.title}} @body={{@model.body}} @author={{user}} />
{{/let}}
```

Here, `this.session.currentUser` is an explicit reference to the component's
instance state, `user` is a local variable introduced by the `#let` helper and
`@model` is a reference to a named argument. They all have obvious semantics.

On the other hand, `BlogPost` and `titleize` are undefined references. The
compiler will assume that they are defined in the surrounding _ambient scope_
at runtime and produce an output like this:

```js
import { precompileTemplate } from '@ember/template-compilation';

precompileTemplate(`{{#let this.session.currentUser as |user|}}
  <BlogPost @title={{titleize @model.title}} @body={{@model.body}} @author={{user}} />
{{/let}}`, {
  moduleName: 'index.hbs',
  strict: true
}); /* => `{
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "meta": { "moduleName": "index.hbs" },
  "scope": () => [BlogPost, titleize]
}` */
```

Again, the specific format here is unimportant and subject to change. The key
here is that the JavaScript code produced by the compiler contains references
(via the `scope` closure in this hypothetical compilation) to the JavaScript
variables `BlogPost` and `titleize` in the surrounding JavaScript scope.

The build tool is responsible for "linking" these undefined references by
putting the compiled JavaScript code inside a JavaScript context where these
variables are defined. Otherwise, depending on the configuration, the undefined
references will either cause a static (build-time) error from the linter,
transpiler (e.g. babel) or packager (e.g. rollup or webpack), or a runtime
`ReferenceError` when the code is evaluated by a JavaScript engine.

This low-level, primitive feature is mainly useful for building user-facing
template import and single-file component features. While this RFC does not
propose a user-facing syntax for these features, here is a hypothetical
template import syntax for the illustrative purposes only:

```hbs
---
import { titleize } from '@ember/template-helpers';
import BlogPost from './components/blog-post';
---

{{#let this.session.currentUser as |user|}}
  <BlogPost @title={{titleize @model.title}} @body={{@model.body}} @author={{user}} />
{{/let}}
```

The build tool can compile this into a JavaScript module like this:

```js
import { createTemplateFactory } from '@ember/template-factory';
import { titleize } from '@ember/template-helpers';
import BlogPost from './components/blog-post';

export default createTemplateFactory({
  "id": "ANJ73B7b",
  "block": "{\"statements\":[\"...\"]}",
  "meta": { "moduleName": "index.hbs" },
  "scope": () => [BlogPost, titleize]
});
```

When this is evaulated by a JavaScript engine, the references in the `scope`
closure will automatically be "linked up" with the imports, and Ember will be
able to reference these values when rendering the template. Note that these
references are _static_–the values are essentially "snapshotted" by the
rendering engine whenever the template is instantiated. Updates to these values
in the JavaScript scope will _not_ be observable by the rendering engine, even
in conjunction with `Ember.set` or `@tracked`.

Optionally, the build tool can choose to restrict the set of allowed ambient
references by suppling an array of available identifiers to the compiler:

```js
import { precompileTemplate } from '@ember/template-compilation';

precompileTemplate(`{{#let this.session.currentUser as |user|}}
  <BlogPost @title={{titleize @model.title}} @body={{@model.body}} @author={{user}} />
{{/let}}`, {
  moduleName: 'index.hbs',
  strict: true,
  scope: ['BlogPost', 'titleize']
});
```

If the template compiler encounters any undefined references outside of this
allowed list, it will throw an error with the appropiate location info. It also
follows that build tools can choose to disable this feature completely by
passing an empty array.

### Keywords

While most items should be imported into scope explicitly, some of the existing
constructs in the language are unimportable will be made available as keywords
instead:

* `action`
* `debugger`
* `each-in`
* `each`
* `has-block-params`
* `has-block`
* `hasBlock`
* `if`
* `in-element`
* `let`
* `link-to` (non-block form curly invocations)
* `loc`
* `log`
* `mount`
* `mut`
* `outlet`
* `query-params`
* `readonly`
* `unbound`
* `unless`
* `with`
* `yield`

These keywords do _not_ have to be imported into scope and will always be
ambiently available.

On the other hand, the following built-in constructs will need to be imported
(the current or proposed import paths in parentheses):

* `array` (`import { array } from '@ember/helper`)
* `concat` (`import { concat } from '@ember/helper`)
* `fn` (`import { fn } from '@ember/helper`)
* `get` (`import { get } from '@ember/helper`)
* `hash` (`import { hash } from '@ember/helper`)
* `on` (`import { on } from '@ember/modifier'`)
* `Input` (`import { Input } from '@ember/component`)
* `LinkTo` (`import { LinkTo } from '@ember/routing`)
* `TextArea` (`import { TextArea } from '@ember/component'`)

In general, built-ins that can be made importable should be imported. The main
difference are that some of the keywords uses internal language features (e.g.
implemented via AST transforms) that requires them to be keywords.

Some of the keywords included in the list are considered legacy, and may be
deprecated in the future via future RFCs. If that happens before strict mode
becomes available in a stable release of Ember, those RFC may propose to drop
support for the legacy keywords in strict mode altogether.

### Deprecations

The following features should be deprecated and removed in non-strict mode:

* Implicit `this` fallback, proposed in [RFC #308](https://github.com/emberjs/rfcs/blob/master/text/0308-deprecate-property-lookup-fallback.md)
* Implicit invocation of argument-less helpers, proposed in [RFC #432](https://github.com/emberjs/rfcs/blob/master/text/0432-contextual-helpers.md)
* Partials, proposed in [RFC #449](https://github.com/emberjs/rfcs/blob/master/text/0449-deprecate-partials.md)

When all of these features are removed, the main difference between non-strict mode
and strict mode will be the precense of globals and the ability to perform
dynamic runtime resolutions.

## How we teach this

Strict mode is intended to become the main way Ember developers author
templates going forward. We anticipate this is going to be a slow transition,
but once the majority of Ember developers have migrated, we expect them to find
it clearer, more intuitive and more productive.

Three of the strict mode restrictions—no implicit `this` fallback, no implicit
invocation of argument-less helpers and no eval—were already proposed in their
respective RFCs. These features will be deprecated in non-strict mode templates and
can be fixed incrementally. We should continue implementing these deprecatios
as already proposed and encouraged adoption. It is quite likely that by the
time strict mode becomes widely available, these deprecations will have already
been implemented and fixed in most Ember applications.

On the other hand, implicit globals and dynamic resolutions are not going away
anytime soon in non-strict mode. These features are intrinsically tied to non-strict
mode, and we expect developers to migrate to template imports or single-file
components when those over time, which would also opt them into strict mode,
when those features become available.

That being said, this RFC does not propose any user-facing changes and there
will be no official, application-level opt-in to strict mode until either
template imports or single-file components exit the experimental phase and
become adopted as official feature in Ember via a future RFC.

Therefore, we do not recommend making any changes to the guides to document the
strict mode semantics at this stage. Instead, the guides should be updated to
feature template imports or single-file components when they become available.

As for the low-level APIs, we should update the API documentation to cover the
new flags (`strict` and `scope`). The documentation should cover the details of
the "ambient scope" feature discussed in this RFC, and emphasize that it is
intended for linking static values such as helpers and components.

## Drawbacks

We could just deprecate without removing `this` fallback and partials, and let
implicit globals and dynamic resolution co-exist with template imports (the
primary consumer of the proposed strict mode). However, this will create a very
confusing compromise and users will not get most of the benefits of having
template imports in the first place. We will also lose out on the opportunity
to improve on the static guarantees in order to build better tools. Leaving
around implicit globals also has the [issues](https://github.com/emberjs/rfcs/blob/master/text/0432-contextual-helpers.md#relationship-with-globals)
discussed in the contextual helpers RFC.

## Alternatives

1. Instead of bundling these into a single "strict mode" opt-in, we could
   allow developers to opt-in to each of these restrictions individually.

   In addition to the teaching and discoverability problems, we will also need
   to build additional tooling and configuration mechanism (`handlebars.json`?)
   for this.

   By adopting these piecemeal, we will also have to define the interaction and
   combined semantics for any possible combinations of these flags, and tooling
   will be unable to take advantage of the improved static guarentees without
   doing a lot of work to account for all these possibilities.

2. Instead of proposing a standalone strict mode, we could just bundle these
   semantics into the templates imports proposal.

   That would make it a very long and complex RFC. In addition, other build
   tools like [ember-cli-htmlbars-inline-precompile](https://github.com/ember-cli/ember-cli-htmlbars-inline-precompile)
   will not be able to adopt the same semantics.

3. Switch to HTML attributes by default in strict mode.

   Today, Glimmer uses a complicated set of huristics to decide if a bound HTML
   "attribute" syntax should indeed be set using `setAttribute` or set as a
   JavaScript property using `element[...] = ...;`. This does not always work
   well in practice, and it causes a lot of confusion and complexity.

   We intend to move to an "attributes syntax always mean attributes" (and use
   modifiers for the rare cases of setting properties). We briefly considered
   grouping that change into the strict mode opt-in, but ultimately decided it
   would be too confusing for strict mode to include such a change. It's better
   to deprecate the feature and make this an app-wide setting.

4. Fix `(action ...)` binding semantics in strict mode.

   Similarly, there are some not ideal semantics issues with `(action ...)`
   around how the function's `this` is bound. We similarly considered fixing it
   in strict mode but ultimately decided it wouldn't be appropriate.

## Unresolved questions

None
