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

This RFC propose to introduce a `template()` function as an alternative for
the `<template>` syntax extension defined in [RFC #779][rfc-779] using
standard JavaScript:

```js
import { template } from "@ember/template-compilation";
import Hello from "my-app/components/hello";

// <template><Hello></template> becomes...
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

[rfc-779]: ./0779-first-class-component-templates.md

## Motivation

[RFC #779](rfc-779) introduced a first-class component syntax feature that has
numerous benefits, among which are:

1. Opt-in to strict mode (RFC #496)
2. Eliminating the need for runtime name-based resolutions
3. Ability to use imported components/helpers/modifiers
4. Ability to interact with the surrounding JavaScript scope

However, one of the drawbacks is that it requires a custom syntax extension
(`<template>`) and a custom file format (`.gjs/.gts`), which requires a build
step and other custom tooling. RFC #779 makes a good case for why this is
necessary and preferable to the alternatives.

However, there remain cases where this drawback is highly undesirable or simply
not acceptable:

1. Until we have implemented good editor integration, the developer experience
   of using `.gjs`/`.gts` could be abysmal (without any syntax highlighting or
   completion at all, etc). Even after we ship good editor integration for the
   mainstream editors, this will probably remain the case for a subset of less
   used editors that we did not or could not prioritize supporting.

2. There may be editors and environments that are impossible for us to support,
   either because they don't have an extension system at all, or those systems
   do not expose the capabilities we need. For example, CodeSandbox, CodPen,
   TypeScript playground, etc.

3. When publishing packages to NPM, it is important and desirable to publish
   only standard `.js` files that does not require further processing by the
   consumer (see: [v2 Addon Format](./0507-embroider-v2-package-format.md)).

4. There are use cases that prefer or require components that are directly
   runnable in the browser without a build step, such as bug report templates,
   runnable code samples, generating components dynamically at runtime, etc.

From here on, we will refer to users and use cases collectively as "standard
JavaScript environments". Under these circumstances, users would be unable to
adopt the `<template>` feature, which means they cannot take advantage of the
numerous benefits unlocked by the feature, such as the strict mode opt-in.

Also, if an addon chose to only provide its template constructs exclusively via
importable modules (as opposed to merging into "App Javascript"), then there is
no easy way to access these template constructs from those environments.

It is not technically impossible to accomplish the same goals in standard
JavaScript environments. After all, the `<template>` feature is specified using
primitives that already exist. Users in standard JavaScript environments can
just use those primitives directly.

However, that is not an ideal outcome. `<template>` is a user-facing feature,
and is poised to (or at least well-positioned to) become the main way Ember
users read, write and reason about components in the next edition of Ember when
the feature is fully rolled out.

On the other hand, the primitives it is based on are very low-level, verbose,
and were more intended as a compilation target than an user-facing authoring
format. Their low-level nature and flexibility also makes it easy to get the
details wrong, such as forgetting to pass the `strictMode: true` flag. It would
be unfortunate if being restricted to the standard JavaScript environment also
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

2. The naming conflicts with the "official" `hbs`. This would have been okay if
   we chose to go that route in RFC #779 and have plans to evolve the official
   version to have the same feature. With RFC #779 favoring the `<template>`
   approach, this is not going to happen and this conflict will now be quite
   confusing, especially since they actually do very different things (e.g.
   the "official" `hbs` does not return a component).

3. It uses a standard JavaScript syntax in a non-standard way, semantics-wise –
   it looks like a JavaScript string but has access to the lexical scope around
   it (without using the `${ ... }` syntax), which can be confusing. It also
   means that it is impossible to provide a runtime implementation using the
   same syntax that works in environments that does not permit a build step.

4. It uses `static template = ` to associate templates to classes. This implies
   a runtime behavior that is actually not true (e.g. TypeScript will believe
   the `template` property exists on the class).

5. Values consumed by the template but is otherwise unused in the rest of the
   file may generate errors/warnings from linters or language servers.

To address these issues, this RFC propose that we introduce a new `template()`
function that servers as a middle ground between the `<template>` language
extension and the low-level primitives.

- It is designed to have the same semantics as the `<template>` feature (such
  as the strict-mode opt-in, returning a template-only component when not
  attached to a class), providing the same high-level programming model for
  authoring components in standard JavaScript environments.

- It requires manually supplying the lexical scope variable bindings.

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
  feature, and uses a static initializer block to associate the template.

  This form will produce an error at runtime if not pre-processed out
  as a correct runtime implementation is impossible.

  Despite the drawbacks mentioned above, this format may still be preferred as
  a transitional tool for early adopters while editor integrations are being
  worked on. It may also be useful for communicating with code snippets in
  platforms where syntax highlighting is not yet available for the custom
  `.gjs` format (such as Discord and GitHub today), but where it's desirable to
  still retain the highlighting for the JavaScript/TypeScript portions.

## Detailed design

This RFC proposes to re-specify the `<template>` language extension into a
de-sugaring into `template()` calls:

1. Top-level declaration

   ```
   import Foo form 'somewhere';

   <template>
     <Foo />
   <template>
   ```

   becomes

   ```js
   import { template } from '@ember/template-compilation';
   import Foo form 'somewhere';

   export default template(`
     <Foo />
   `, () => ({ Foo }));
   ```

2. Expression

   ```
   import Foo form 'somewhere';

   export const Bar = <template><Foo /><template>;
   ```

   becomes

   ```js
   import { template } from '@ember/template-compilation';
   import Foo form 'somewhere';

   export const Bar = template(`<Foo />`, () => ({ Foo }));
   ```

3. Class

   ```
   import Foo form 'somewhere';

   export default class Bar {
     <template><Foo /><template>
   }
   ```

   becomes

   ```js
   import { template } from '@ember/template-compilation';
   import Foo form 'somewhere';

   export default class Bar {
     static {
       template(`<Foo />`, () => ({ Foo }), this);
     }
   }
   ```

   **Note**: In static initializer block isn't required here (though it is a
   standard JavaScript feature). The `template()` function can wrap around the
   class, or be applied after the class has been declared.

In these small snippets, the scope bindings may look very verbose compared to
the size of the templates, but in real-world templates, the ratio will improve.

**TBD**: I think we should amend the v2 addon spec to make this the recommended
publishing format.

**Note**: Whether we actually implement `<template>` using this desugaring is
an internal implementation detail. However, the semantics should work and users
should be able to reason about `<template>` with this desugaring in mind.

### Tagged Template Literals

1. Top-level declaration

   ```
   import Foo form 'somewhere';

   <template>
     <Foo />
   <template>
   ```

   becomes

   ```js
   import { template } from '@ember/template-compilation';
   import Foo form 'somewhere';

   export default template`<Foo />`;
   ```

2. Expression

   ```
   import Foo form 'somewhere';

   export const Bar = template`<Foo />`;
   ```

   becomes

   ```js
   import { template } from '@ember/template-compilation';
   import Foo form 'somewhere';

   export const Bar = template`<Foo />`;
   ```

3. Class

   ```
   import Foo form 'somewhere';

   export default class Bar {
     <template><Foo /><template>
   }
   ```

   becomes

   ```js
   import { template } from '@ember/template-compilation';
   import Foo form 'somewhere';

   export default class Bar {
     static { template`<Foo />` }
   }
   ```

**TBD**

## How we teach this

**TBD**: While the `template()` function is a bit cumbersome to use in practice
I think what it does is in fact quite easy to understand and teach. It may
actually help with teaching `<template>` if we can show what it desugars into.

## Drawbacks

**TBD**: Having the ability to supply the variable binding closure manually may
open it up to misuse (i.e. the closure does more than just simply capturing the
surrounding lexical scope, or does so in ways we didn't intend). However, this
is not a new problem in a sense, as the primitives already exists, and the
status quo is "if you cannot use `<template>`, use the primitives directly.
Also, with default helper manager, you can pretty much do anything along those
lines anyway with an inline closure that you invoke inside the template.

## Alternatives

**TBD**

```js
export @template(`<Hello />`, () => ({ Hello }) class Bar {
}
```

## Unresolved questions

**TBD**
