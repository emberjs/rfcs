---
stage: accepted
start-date: # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite:
---

<!---
Directions for above:

stage: Leave as is
start-date: Fill in with today's date, 2032-12-01T00:00:00.000Z
release-date: Leave as is
release-versions: Leave as is
teams: Include only the [team(s)](README.md#relevant-teams) for which this RFC applies
prs:
  accepted: Fill this in with the URL for the Proposal RFC PR
project-link: Leave as is
suite: Leave as is
-->

# JS Representation of Template Tag

## Summary

Formalize a Javascript-spec-compliant representation of template tag.

## Motivation

The goal of this RFC is to simplify the plain-Javascript representation of the Template Tag (aka "first-class component templates") feature in order to:

- reduce the number and complexity of API calls required to represent a component
- efficiently coordinate between different layers of build-time and static-analysis tooling that need to use preprocessing to understand our GJS syntax extensions.
- avoid exposing a "bare template" as a user-visible value
- provide a declarative way to opt-in to run-time (as opposed to build-time) template compilation.

As an illustrative example, currently this template tag expression:

```js
let x = <template>Hello {{ message }}</template>;
```

Has the plain javascript representation:

```js
import { precompileTemplate } from "@ember/component";
import templateOnlyComponent from "@ember/component/template-only";
import { setComponentTemplate } from "@ember/component";
let x = setComponentTemplate(
  precompileTemplate("Hello {{message}}", {
    strictMode: true,
    scope: () => ({ message }),
  }),
  templateOnlyComponent()
);
```

This RFC proposes simplifying the above case to:

```js
import { template } from "@ember/template-compiler";
let x = template("Hello {{message}}", {
  scope: () => ({ message }),
});
```

As a second illustrative example, currently this class member template:

```js
class Example extends Component {
  <template>Hello {{message}}</template>
}
```

Has the plain javascript representation:

```js
import { precompileTemplate } from "@ember/component";
import { setComponentTemplate } from "@ember/component";
class Example extends Component {}
setComponentTemplate(
  precompileTemplate("Hello {{message}}", {
    strictMode: true,
    scope: () => ({ message }),
  }),
  Example
);
```

This RFC proposes simplifying the above case to:

```js
import { template } from "@ember/template-compiler";
class Example extends Component {
  static {
    template(
      "Hello {{message}}",
      {
        scope: () => ({ message }),
      },
      this
    );
  }
}
```

## Detailed design

This RFC introduces two new importable APIs:
```js
// The ahead-of-time template compiler:
import { template } from "@ember/template-compiler";

// The runtime template compiler:
import { template } from "@ember/template-compiler/runtime";
```

They are intended to be drop-in replacements for each other *except* for the differences summarized in this table:


|  | Template Contents | Scope Param | Syntax Errors | Payload |
| --- | --- | -- | -- | -- |
| Ahead-of-Time| Restricted to literals | Restricted to a few explicitly-allowed forms | Stops your build | Smaller & Faster
| Runtime | Unrestricted | Unrestricted | Can by caught at runtime | Larger & Slower


By putting these two implementations in different importable modules, the problem of "how do you opt-in to including the template compiler in your app?" goes away. If you import it, you will have it, if you don't, you won't.

The remainder of this design only uses examples with the ahead-of-time template compiler, because everything about the runtime template compiler's API is the same.

### Type Signature

```ts
function template(
  templateContent: string, 
  params?: Params, 
  backingClass?: object
): TheComponent;

// This is the actual invokable component. Needs discussion with typescript team to formalize the correct type here and make sure the important inferrence cases work.
type TheComponent = TODO;

interface Params {
  strict?: boolean;
  scope?: Scope
  moduleName?: string;
}

type Scope =
  | (() => Record<string, unknown>)
  | ((local: string, instance: any) => any);

```

### Strict defaults to true

Unlike `precompileTemplate`, our `strict` param defaults to true instead of false if it's not provided. This is aligned with the expectation that our main programming model is moving everyone toward handlebars strict mode by default.

This also addresses the naming confusing between earlier RFCs (which used "strict") and the implementations in the ecosystem (which used "strictMode").

### Always Returns a Component

A key difference between `precompileTemplate` and our new `template` is that its return value is always a _component_, never a "bare template". In this sense, the implementation combines the jobs of `precompileTemplate` and `setComponentTemplate`.

Bare templates are a historical concept that we'd like to move away from, in order to have fewer things to teach.

When the optional `backingClass` argument is passed, the return value is that backing class, with the template associated, just like `setComponentTemplate`. When the `backingClass` argument is not provided, it creates and returns a new template-only component.

> *Aren't route templates "bare templates"? What about them?<br>*
> Yes, this RFC deliberately doesn't say anything about route templates. We expect a future routing RFC to use components to express what today's route templates express. This RFC also doesn't deprecate `precompileTemplate` yet -- although that will clearly be a good goal _after_ a new routing design addresses route templates.

### Scope Parameter

The scope parameter exists for the same reason as is exists in `precompileTemplate`: it gives the template access to things from the surrounding Javascript scope.

We accept two different forms, distinguished by function arity:

1. `() => Record<string, unknown>` is exactly like the existing `scope` in precompileTemplate. The values in the returned object are the lexically scoped local variables available to the template. 
2. `(local: string, instance: any) => any` is an alternative form that does per-local-name lookup.

The first form is the preferred choice for addons to publish to NPM and any other long-term stable source code. It makes the data flow statically apparent to all Javascript-aware tooling. To produce this form, build tooling needs to do a full parse of the template to discover exactly what upvars are discovered in the `<template></template>` tag, and intersect them with the set of bindings in Javascript scope.

The second form is the preferred format to pass from a GJS preprocessor to other Javascript toolchains that don't understand GJS (like Babel or SWC). The typical usage would look like:

```js
template("Hello {{message}}", { scope(c8bb1dc1e509aa6b, fca6cbb3e151d53f) => eval(c8bb1dc1e509aa6b) }});
```

> *Why the weird local variable names*?<br>
> In order to let the `eval` access anything in scope, we need to avoid shadowing any existing names. Since this is supposed to be the cheap transformation that doesn't require lexical analysis, we'd rather avoid the collision by picking well-known randomized identifiers.


This is cheap to produce from `<template>` tag because you don't need to do any parsing of the template contents. It's the minimal transformation that gets you from GJS to valid Javascript. This would serve the same role that `[__GLIMMER_TEMPLATE(...)]` currently serves in the implementation of ember-template-imports, but unlike that syntax it can actually have a working runtime implementation, because the eval can grab local variables from scope as needed.

> *Javascript Semantics what now?*<br>
> If we only convert the contents of a template tag to a Javascript string literal, we're hiding an importing fact: the template may actually be consuming some bindings from the surrounding scope. `eval()` is *also* allowed to consume bindings from the surrounding scope, so its presence means that a runtime implementation can actually work.

>*eval seems bad, what about Content Security Policy?*<br>
> Nobody needs to actually *run* the eval. This is a communication format between layers of build tooling. You *can* run it, if you're making something like an interactive development sandbox. But that is not the typical case.


### Syntactic Restrictions

The runtime template compiler has no syntactic restrictions.

The ahead-of-time template compiler has syntactic restrictions on `templateContents` and `params.scope`. 

`templateContents` must be one of:
 - a string literal
 - a template literal with no expressions

 If provided, `params.scope` must be one of the following:
  - an arrow function expression or function expression
    - with body containing either
      - an expression
      - or a block statement that contains exactly one return statement
    - where the return value is an object literal
      - whose properties are all non-computed
      - whose values are all identifiers
  - the exact object method `scope(c8bb1dc1e509aa6b, fca6cbb3e151d53f){ return eval(c8bb1dc1e509aa6b) }`
  - the exact object property `scope: function(c8bb1dc1e509aa6b, fca6cbb3e151d53f) { return eval(c8bb1dc1e509aa6b) }`
  - the exact object property `scope: (c8bb1dc1e509aa6b, fca6cbb3e151d53f) => eval(c8bb1dc1e509aa6b)`. 


## How we teach this

> What names and terminology work best for these concepts and why? How is this
> idea best presented? As a continuation of existing Ember patterns, or as a
> wholly new one?

> Would the acceptance of this proposal mean the Ember guides must be
> re-organized or altered? Does it change how Ember is taught to new users
> at any level?

> How should this feature be introduced and taught to existing Ember
> users?

## Drawbacks

> Why should we _not_ do this? Please consider the impact on teaching Ember,
> on the integration of this feature with other existing and planned features,
> on the impact of the API churn on existing apps, etc.

> There are tradeoffs to choosing any path, please attempt to identify them here.

## Alternatives

This RFC builds off the proposal in 
https://github.com/emberjs/rfcs/pull/813.

The main difference is that RFC 813 offered a form:

```
template`<Foo />`
```
that converts `<template>` into valid JS syntax *without* working JS semantics. 

## Unresolved questions

> Optional, but suggested for first drafts. What parts of the design are still
> TBD?

 - What TS type to use for the component

 - Is `@ember/template-compiler` the right import path?

 - TODO: audit the other existing arguments we pass through both precompileTemplate and the underlying lower-level template compiler and decide which to keep.



