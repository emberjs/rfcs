---
Stage: Accepted
Start Date: 2021-12-03
Release Date: Unreleased
Release Versions:
  ember-source: vX.Y.Z
  ember-data: vX.Y.Z
Relevant Team(s): Ember.js, Learning, Ember CLI
RFC PR: https://github.com/emberjs/rfcs/pull/779
---


# First-Class Component Templates


## Summary

Adopt `<template>` tags as a format for making component templates first-class participants in JavaScript and TypeScript with [strict mode][rfc-0496] template semantics. In support of the new syntax, adopt new custom JavaScript and TypeScript files with the extensions `.gjs` and `.gts` respectively.

First-class component templates address a number of pain points in today’s component authoring world, and provide a number of new capabilities to Ember and Glimmer users:

- accessing local JavaScript values with no ceremony and no backing class, enabling much easier use of existing JavaScript ecosystem tools, including especially styling libraries—standard [CSS Modules][css-modules] will “just work,” for example

- authoring more than one component in a single file, where colocation makes sense—and thereby providing more control over a component’s public API

- likewise authoring locally-scoped helpers, modifiers, and other JavaScript functionality

First-class component templates offer these new capabilities while not only maintaining but *improving* Ember’s long-standing commitment to integrated testing, in that it allows app and test code to share a single authoring paradigm—substantially simplifying our teaching story. Similarly, it preserves Ember’s long-standing commitment to treating JavaScript and HTML (and CSS!) as distinctive concerns which, however closely related, are not the *same*.

<details><summary>Full-fledged example showing how this might work in practice</summary>

Two notes:

- For this and all the examples in the RFC, I assume [RFC #0757: Default Modifier Manager][rfc-0757] for simplicity, but it does not meaningfully change *this* proposal.

- The syntax highlighting here is a mess… but that's because GitHub still doesn't have good highlighting for *decorators*. Samples which have `<template>` but *not* `@tracked` actually already highlight decently well.

[rfc-0757]: https://github.com/emberjs/rfcs/pull/757

```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';

const Greet = <template>
  <p>Hello, {{@name}}!</p>
</template>

class SetUsername extends Component {
  @tracked name = '';

  updateName = ({ target: { value } }) => {
    this.name = value;
  }

  saveName = (submitEvent) => {
    submitEvent.preventDefault();
    this.args.onSaveName(this.name);
  };

  <template>
    <form {{on "submit" this.saveName}}>
      <label for='name'>Set username:</label>
      <input
        id='name'
        value={{this.value}}
        {{on "input" this.updateName}}
      />
      <button type='submit' disabled={{eq this.value.length 0}}>
        Generate
      </button>
    </form>
  </template>
}

function replaceLocation(el, { with: newUrl }) {
  el.contentWindow.location.replace(newUrl);
}

export default class GenerateAvatar extends Component {
  @tracked name = "";

  get previewUrl() {
    return `http://www.example.com/avatars/${name}`;
  }

  updateName = (newName) => {
    this.name = newName;
  };

  <template>
    <Greet @name={{this.name}} />
    <SetUsername
      @name={{this.name}}
      @onSaveName={{this.updateName}}
    />
    
    {{#if (gt 0 this.name.length)}}
      <iframe
        title='Preview'
        {{replaceLocation with=this.previewUrl}}
      >
    {{/if}}
  </template>
}
```

</details>


### Contents <!-- omit in toc -->

- [Summary](#summary)
- [Motivation](#motivation)
  - [Namespaces and Modules](#namespaces-and-modules)
  - [Scope](#scope)
  - [Ecosystem integration](#ecosystem-integration)
  - [Testing](#testing)
  - [The solution](#the-solution)
  - [Constraints](#constraints)
- [Detailed design](#detailed-design)
  - [Compilation](#compilation)
    - [Standalone](#standalone)
    - [Bound to a name](#bound-to-a-name)
    - [Class body](#class-body)
  - [Performance](#performance)
  - [Interop](#interop)
    - [Named exports](#named-exports)
    - [Non-colocated templates](#non-colocated-templates)
  - [The “prelude”](#the-prelude)
  - [Tooling](#tooling)
    - [Syntax highlighting](#syntax-highlighting)
    - [Blueprints](#blueprints)
    - [Linting and formatting](#linting-and-formatting)
    - [Language server support](#language-server-support)
    - [Codemod](#codemod)
  - [TypeScript](#typescript)
  - [Custom file extension](#custom-file-extension)
- [Transition path](#transition-path)
- [How we teach this](#how-we-teach-this)
  - [Guides](#guides)
    - [Tutorial](#tutorial)
    - [Core Concepts: Components](#core-concepts-components)
  - [API Docs](#api-docs)
  - [Existing Ember users](#existing-ember-users)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [TypeScript signature](#typescript-signature)
  - [Distinguishing class-backed and template-only components](#distinguishing-class-backed-and-template-only-components)
  - [Alternative syntaxes](#alternative-syntaxes)
    - [Imports-only](#imports-only)
    - [SFCs](#sfcs)
    - [Template literals (`hbs`)](#template-literals-hbs)
- [Unresolved questions](#unresolved-questions)


## Motivation

Today, authors of Ember and Glimmer apps and libraries must author their templates and JavaScript in separate `.hbs` and `.js` files, and the templates exist in a “resolution” mode where every component, helper, and modifier exists in a single global namespace. This has a number of significant downsides. What’s more, there are significant new capabilities for Ember and Glimmer authors made available by embracing JavaScript scope—while keeping our commitments to testing and separation of concerns.[^original-primitives-rfc]

[^original-primitives-rfc]: See also [the SFC & Template Import Primitives RFC](https://github.com/emberjs/rfcs/blob/tomdale/template-primitives/text/000-sfc-and-template-import-primitives.md), which described the motivation for implementing the primitives on which this proposal will build.


### Namespaces and Modules

First, because of the global namespace, name conflicts are common, and avoiding them requires either manually namespacing components or using (now-deprecated) experimental tools like [`ember-holy-futuristic-template-namespacing-batman`](https://github.com/rwjblue/ember-holy-futuristic-template-namespacing-batman). But:

- Manually namespacing is clunky and does not actually *guarantee* there won't be conflicts. Combined with the way addons typically supply their components, helpers, and modifiers into the app namespace, name conflicts are sometimes unavoidable.

- Even the workaround via `ember-holy-futuristic-template-namespacing-batman` requires using different names for modules in Ember than their Node package name when the Node package uses [npm scopes][scopes]. (This was one of the original motivations for exploring a design which leverages JavaScript modules, in fact!) Since our resolution modes must ultimately deal in JavaScript terms, we are in the position of always potentially being one ecosystem shift away from another syntax conflict with template-language-only designs for managing scope.

- It requires our tooling to understand Ember's resolution rules, and our tooling cannot take advantage of existing ecosystem tooling. Our language servers, for example, have to more or less re-implement Ember’s resolver themselves.

- There is a substantial performance cost to dynamically resolving components by name at runtime. This can be mitigated by using the combination of something like `ember-holy-futuristic-template-namespacing-batman` with the [strict resolver][strict-resolver], but the strict resolver is not standard—and really cannot be without something like this proposal.[^strict-resolver-rfc]

- It is extremely unpleasant (though, strictly speaking, *possible* as of Ember 3.25[^verbose-local]) to introduce a component, helper, or modifier which is not in the global namespace. See the next section, [**Scope**](#scope), for further on this.

The global namespace also comes with overhead for our teaching story by introducing a layer of “magic”: people just have to memorize that a file with a default export in a given location *automagically* is available with a given name. This is just a “bare fact”: there is nothing to connect it to in terms of a developers’ existing JavaScript or HTML knowledge.

[strict-resolver]: https://github.com/stefanpenner/ember-strict-resolver

These problems are all well-solved already, using the JavaScript modules spec (or "ESM", for ECMAScript Modules). Today, however Ember developers cannot take advantage of those or the tooling which understands them!

[^strict-resolver-rfc]: See the still-open [RFC #0683][rfc-0683] for a discussion of the full set of concerns involved in resolution, which include but are not limited to the template concerns addressed here.

[rfc-0683]: https://github.com/emberjs/rfcs/pull/683

[^verbose-local]: In all cases, doing so requires introducing a backing class to make the value available to the template *or* writing Ember's strict mode template syntax manually (which is error-prone and extremely verbose: it is designed as an output format, not an authoring format).


### Scope

Second, and closely related to the global namespace problem: there is presently no good way for users to introduce or use locally-scoped code. Every component, helper, and modifier must live in its own file, and be globally available—even if it is meant to be used privately. Where JavaScript modules provide users to control their public APIs in terms of `export`s, Ember apps largely cannot take advantage of exports for anything which interacts with the template layer.

In practice, this has a number of knock-on effects for Ember code.

First, components tend to grow without bound, because the equivalent of the "extract method" or "extract into new class" refactors (which we commonly use on the JS side) end up with two downsides:

- they make the newly-extracted components available to the whole app, even if the concern is private to that component
- the require an entirely new file, which is friction both for the creation and the use/understanding of a given view

Second, users also often introduce classes with actions or getters where a simple [function-based helper][rfc-0756] would do, because that is the only way to provide a non-global function. (I show this by example in [**How We Teach This: Guides: Tutorial: Reusable Components**](#reusable-components) below.)

Third, it likewise incentivizes the use of the ember-render-modifiers with backing classes, rather than custom modifiers, because the behavior can then be scoped to that module—whereas, again, a custom modifier would be in global scope. This in turn makes it easy for users to miss the helpful separation of concerns which custom modifiers enable.

[rfc-0756]: https://github.com/emberjs/rfcs/pull/756

Over time, these all lead to a proliferation of backing classes which are only present to work around the fact that we have no *other* way to provide non-global scope for our components. These classes in turn tend to act as “state attractors,” leading to an unnecessary proliferation of state throughout an app or addon.

[scopes]: https://docs.npmjs.com/cli/v8/using-npm/scope

  
### Ecosystem integration

Tools which assume they will be used in JavaScript contexts more or less don’t work with our templates today, because the templates have no way to access them. Think of CSS tools like [CSS Modules][css-modules], which is widely used in the Ember ecosystem via [Ember CSS Modules][e-css-m]: our current implementation has to jump through many hoops and do many hacks to work at all. These problems are fundamental to the current model. A format which makes JavaScript values available in template scope would let us drop *all* of that special sauce—and this goes for *all* such JavaScript-side tooling.[^other-css-tools]

[css-modules]: https://github.com/css-modules/css-modules
[e-css-m]: https://github.com/salsify/ember-css-modules

[^other-css-tools]: The same applies to all other similar tools, e.g. [Emotion][emotion], [Styled Components][styled-components], [vanilla-extract][vanilla-extract]: none of them work out of the box with our current design. Whatever anyone’s personal opinions on these specific, they’re potentially-valuable tools which are barely or not at all usable in Ember today.

[emotion]: https://emotion.sh/docs/@emotion/css
[styled-components]: https://styled-components.com
[vanilla-extract]: https://vanilla-extract.style


### Testing

Finally, the authoring format for tests and the authoring format for app code today is completely different. A test can render a component by calling the `render()` function from `@ember/test-helpers` and passing it a Handlebars string.[^testing-rfc] App code *cannot* do this or anything like it. This has teaching overhead: we both *can* do things in tests we *cannot* do in app code, raising the obvious “but why not?”; and we also *must* do things in tests we do not *need* to do in app code.

Additionally, introducing test-only components is quite painful, requiring use of the `this.owner.register()` functionality, and therefore requiring users to understand at least some of Ember’s custom runtime resolution (as well as learning a microsyntax for it[^microsyntax]). What's more, authoring a *template* for a test-only component is undocumented and is also entirely unlike the story for authoring templates for app components.

[^testing-rfc]: The test helper `render()` also does not actually render components today—but the *mental model* is that it does. Here and below I assume a forthcoming RFC which will allow `render` to work not only with templates (the _status quo_) but also with components. This will be an independent change which helps eliminate a number of quirks in the testing infrastructure today as well as make it more TypeScript friendly, but it complements *this* RFC by allowing local definition of tests.

[^microsyntax]: made that much more bespoke since [RFC #0585](https://emberjs.github.io/rfcs/0585-improved-ember-registry-apis.html) is accepted but not yet implemented


### The solution

To address these problems, the Ember community [proposed primitives][rfc-0454] which unlocked experimentation in this space and [defined the semantics of “strict” templates which use those primitives][rfc-0496] and [made modifers and helpers first-class citizens of templates][rfc-0432]. Now, with a history of having done that experimentation—with [GlimmerX][glimmerx] and [ember-template-imports][eti]—and having had *many* discussions about the trade-offs over the years, it’s time to ship a proposal which resolves these questions: **first-class component templates**.

[rfc-0454]: https://github.com/emberjs/rfcs/pull/454
[rfc-0496]: https://emberjs.github.io/rfcs/0496-handlebars-strict-mode.html
[rfc-0432]: https://emberjs.github.io/rfcs/0432-contextual-helpers.html
[glimmerx]: https://glimmerjs.github.io/glimmer-experimental/
[eti]: https://github.com/ember-template-imports/ember-template-imports

In this new world, templates are authored in JavaScript files with a `<template>` tag. Templates defined this way have normal Glimmer template semantics, but instead of using a runtime resolution strategy, they have access to values in JavaScript scope, which means they can just use normal JavaScript imports. What's more, they can define other local components, helpers, or modifiers and export them or not as makes sense. They can do the same kind of extraction refactors they do with JavaScript or CSS. And other tools from the JavaScript ecosystem “just work”—from custom CSS tooling to GraphQL queries authored with [Apollo Client’s `graphql` template strings][graphql] and anything else the ecosystem comes up with.

[graphql]: https://www.apollographql.com/docs/react/data/queries/

At the same time, since the body of a template defined with a `<template>` tag has all the same rules as Glimmer templates do today, this new authoring format keeps all the goodness of today’s clear separation of concerns between HTML and JavaScript and CSS. That means it continues to empower developers who are HTML and CSS experts and reach for JavaScript only secondarily. Indeed, the design goes out of its way to make HTML/Handlebars-only files feel like first-class citizens.

Finally, introducing `<template>` completely unifies the story between app and test code: in this new world, introducing a test-only component is as simple as introducing any other component in the same file as an existing component.

In sum, `<template>` resolves each problem outlined above, and introduces new capabilities to boot.


### Constraints

There are a number of solutions which could address these needs and add these capabilities. This RFC proposes `<template>` out of all the possible options because I take the following constraints as guiding the design decision (and the ordering here is purposeful—items earlier in the list I judge to be more important than items later in the list):

1. Our choice of design **must not regress our ability to write tests**, and if it is possible to *improve* our testing story, we should take the opportunity to do so.

2. In the absence of hard technical constraints forbidding it, we should **prefer the solution which has the best story for teaching**—at all levels, including beginners but also supporting advanced users. In particular, this means that we should value both *progressive disclosure of complexity* and the *principle of least surprise*, and that we may need to weight them against each other, but that we should pay particular attention when they agree.

3. This design must cleanly interoperate with existing Ember code bases. That is, adopting this **must not require users to migrate their entire code base at once**.

4. We should **prefer a design which provides more flexibility** to end users over a design which provides less.

While it is certainly possible to differ with these constraints *a priori*—reevaluating constraints is, in a very real sense, [how we got to this very RFC](https://github.com/emberjs/rfcs/pull/367#issuecomment-423839940)—we also run the risk of paralysis if we *continually* reevaluate from first principles. More challenging is inevitable disagreement about how we *weight* these constraints. On that front, there is no possibility of *final* agreement, but we should commit to some ordering for the purposes of this design so that the rest of it can proceed on the same terms.


## Detailed design

Introduce a new high-level syntax, the `<template>` tag, which is syntactical sugar for `setComponentTemplate` and `precompileTemplate`, in conjunction with the existing Ember and Glimmer `Component` classes and the special template-only component class returned by the `templateOnlyComponent` default export from `@ember/component/template-only`.

There are three distinct, legal forms for this compilation:

- a standalone `<template>` at the top level of a module
- a `<template>` bound to a name
- in the body of a component class

For a discussion of the `setComponentTemplate` and `templateOnlyComponent` primitives, see [RFC #0481][rfc-0481]; for discussion of the `precompileTemplate` primitive, see [RFC #0496][rfc-0496]. This discussion will *assume* rather than *define* those. Additionally, I leave aside here the further build-time passes which transform `precompileTemplate` invocations into a precompiled template in “wire format” ready for use by the Glimmer VM, as that is not affected by the authoring format.

[rfc-0481]: https://emberjs.github.io/rfcs/0481-component-templates-co-location.html


### Compilation

The value produced by authoring a `<template>` is a *JavaScript value*, and accordingly may be exported, bound to other values, passed as an argument to a function or set as a value on a class, and so on. However, that value is not *dynamic*. Instead, it is compiled statically to a format targeting the Glimmer VM at compile time, such that even the `precompileTemplate` invocations are removed in favor of the wire format, which itself may be further optimized or changed in the future.

Therefore, in normal app or addon code, it is nonsensical to use it with a `let` binding: changing the value bound to the `let` will *not* result in Ember’s reevaluating anything which *uses* that value: the “scope” of a template is only ever computed once, for performance reasons.

A function may of course return different components based on its arguments, etc.; but such a function will not be “automatically” re-executed unless the function consumes tracked properties. This is just applying the standard auto-tracking semantics to functions which return components, which is possible *today*. I discuss below the performance pitfalls of doing this inline, and the corresponding guidance we should provide.

Apps or addons which want to compile arbitrary components at runtime are the exception to static component definition as described here. Most apps and addons will *not* want to do this, because it is expensive and slow and also a security risk in that it allows arbitrary code execution within your app. However, there *are* good use cases, e.g. dynamic online environments like the [GlimmerX playground][glimmerx-playground] or the [Limber Editor][limber], or documentation tooling like [Storybook][storybook].

These kinds of apps and integrations can integrate the template compiler as a runtime dependency and build new templates on the fly. However, the details of doing that are unrelated to providing first-class component templates and do not change as a result of this RFC. The scope remains static for any given `<template>` declaration after compilation; the difference there is that they are intentionally re-executing the compilation step itself.

[glimmerx-playground]: https://glimmerjs.github.io/glimmer-experimental/
[limber]: https://limber.glimdown.com/
[storybook]: https://storybook.js.org


#### Standalone

The compiled output for a top-level `<template>` tag is a default export. This means that the very common case of having a simple template-only component looks basically just like HTML, wrapped in `<template>`, helping us provide a strong *progressive disclosure of complexity* flow to our design and our pedagogy. It also means that the basic code Ember developers use today changes very little for the most basic version of the new format. Given this input:

```js
<template>
  <p>Hello, {{@name}}!</p>
</template>
```

The compiled output is:

```js
import { precompileTemplate } from '@ember/template-compilation';
import { setComponentTemplate } from '@ember/component';
import templateOnlyComponent from '@ember/component/template-only';

export default setComponentTemplate(
  precompileTemplate(`
  <p>Hello, {{@name}}!</p>
  `,
    {
      isStrict: true,
    }
  ),
  templateOnlyComponent()
);
```

If the `<template>` references values in scope, they will be included in an object with a `scope` argument (shown here using the current implementation of the underlying primitives). Thus, this definition—

```js
function isBirthday(dateOfBirth) {
  const today = new Date();
  return (
    today.getMonth() === dateOfBirth.getMonth() &&
    today.getDate() === dateOfBirth.getDate()
  );
}

<template>
  <p>Hello, {{@name}}!</p>
  {{#if (isBirthday @dateOfBirth)}}
    <p>Happy birthday! 🎈</p>
  {{/if}}
</template>
```

—compiles to this output:

```js
import { precompileTemplate } from '@ember/template-compilation';
import { setComponentTemplate } from '@ember/component';
import templateOnlyComponent from '@ember/component/template-only';

function isBirthday(dateOfBirth) {
  const today = new Date();
  return (
    today.getMonth() === dateOfBirth.getMonth() &&
    today.getDate() === dateOfBirth.getDate()
  );
}

export default setComponentTemplate(
  precompileTemplate(`
  <p>Hello, {{@name}}!</p>
  {{#if (isBirthday @dateOfBirth)}}
    <p>Happy birthday! 🎈</p>
  {{/if}}
  `,
    {
      isStrict: true,
      scope: () => ({ isBirthday }),
    }
  ),
  templateOnlyComponent()
);
```

Since the values in scope use normal JavaScript semantics, this means that imports also “just work”. Thus, if we extracted `isBirthday` into a separate file for reuse elsewhere, we could import and use it like this:

```js
import { isBirthday } from '../utils/user';

<template>
  <p>Hello, {{@name}}!</p>
  {{#if (isBirthday @dateOfBirth)}}
    <p>Happy birthday! 🎈</p>
  {{/if}}
</template>
```

The compiled output would be just the same as before, save using the imported value:

```js
import { isBirthday } from '../utils/user';
import { precompileTemplate } from '@ember/template-compilation';
import { setComponentTemplate } from '@ember/component';
import templateOnlyComponent from '@ember/component/template-only';

export default setComponentTemplate(
  precompileTemplate(`
  <p>Hello, {{@name}}!</p>
  {{#if (isBirthday @dateOfBirth)}}
    <p>Happy birthday! 🎈</p>
  {{/if}}
  `,
    {
      isStrict: true,
      scope: () => ({ isBirthday }),
    }
  ),
  templateOnlyComponent()
);
```

Since the compiled output is a default export, it is a static error to have multiple top-level (i.e. not bound to a name) `<template>`s in a file—because it is a static error to have multiple `export default` statements in a JavaScript file. We should provide a lint rule to error on this case, rather than letting it fail at build or runtime.


#### Bound to a name

A standalone first-class template can also be bound to a name in the module. This allows users to provide locally-scoped modules as well as a single default export, as well as to use modules as a way of grouping related functionality or hiding private functionality while still being able to refactor and extract common code. Given this input:

```js
const Greet = <template>
  <p>Hello, {{@name}}!</p>
</template>
```

The compiled output is:

```js
import { precompileTemplate } from '@ember/template-compilation';
import { setComponentTemplate } from '@ember/component';
import templateOnlyComponent from '@ember/component/template-only';

const Greet = setComponentTemplate(
  precompileTemplate(`
  <p>Hello, {{@name}}!</p>
  `,
    {
      isStrict: true
    }
  ),
  templateOnlyComponent()
);
```

Values referenced from the surrounding scope are included in exactly the same way as with the standalone top-level declaration.

Notice that this allows for a host of convenient (and likely common!) new ways of providing a group of related components. For example:

- Genuinely private components could be authored within a file which does not export them, and only exports the public API.
- Components can be authored in their own files as default exports, and then importing them and re-exporting them as a namespace from an entry-point module.
- A namespace export allows an library to supply both a default export as its primary entry point and a series of related components within the same module.

No doubt there are many other such useful patterns which will emerge organically here as they have across the broader JS ecosystem.

Users *should* never reassign the result of binding a template, because Ember will never reevaluate if the name is re-bound later. (Even if we wanted to do that, it would be difficult at best: nothing would notify Ember that it *should* re-evaluate that value!) We should introduce a lint rule forbidding reassignment of a `<template>` to a binding to prevent that confusion.


#### Class body

The compilation output with a class-backed component is similar, but instead of using `templateOnlyComponent`, it uses the backing class. Given this component:

```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';

class SetUsername extends Component {
  @tracked name = '';

  updateName = ({ target: { value } }) => {
    this.name = value;
  }

  saveName = (submitEvent) => {
    submitEvent.preventDefault();
    this.args.onSaveName(this.name);
  };

  <template>
    <form {{on "submit" this.saveName}}>
      <label for='name'>Set username:</label>
      <input
        id='name'
        value={{this.value}}
        {{on "input" this.updateName}}
      />
      <button type='submit' disabled={{eq this.value.length 0}}>
        Generate
      </button>
    </form>
  </template>
}
```

The compiled output is:

```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { precompileTemplate } from '@ember/template-compilation';
import { setComponentTemplate } from '@ember/component';

class SetUsername extends Component {
  @tracked name = '';

  updateName = ({ target: { value } }) => {
    this.name = value;
  }

  saveName = (submitEvent) => {
    submitEvent.preventDefault();
    this.args.onSaveName(this.name);
  };
}

setComponentTemplate(
  precompileTemplate(`
    <form {{on "submit" this.saveName}}>
      <label for='name'>Set username:</label>
      <input
        id='name'
        value={{this.value}}
        {{on "input" this.updateName}}
      />
      <button type='submit' disabled={{eq this.value.length 0}}>
        Generate
      </button>
    </form>
  `,
    {
      isStrict: true,
    }
  ),
  SetUsername
)
```

##### Private class fields

In the present design of the template compilation primitives, a template cannot access private fields from the backing class. That is, the following *will not work*:

```js
import Component from '@glimmer/component';

class Example extends Component {
  #aField = true;

  <template>
    <p>The value of the field is: {{this.#aField}}</p>
  </template>
}
```

That is because the compilation output does *not* embed the template in the class' body in any way, but instead associates it *externally* to the class—but private class fields are only accessible within the body of the class itself, per the ECMAScript spec. While we could invest time to change the implementation to avoid this, it is not generally a problem. This is rarely a problem: the only way to get direct access a component instance is to `{{yield this}}` in a component template. For managing privacy, developers should choose to `yield` public API instead (e.g. via a getter, or using `hash` or using a set of positional parameters).

This is a real gap, which we could address in a future RFC. Notably, however, it is *not specific to this proposal*, but applies to *all* proposals built on the current primitives.


### Performance

In normal app code, authors should not introduce component definitions `<template>`s in contexts where they will be “re-executed,” e.g. in a function body. It is technically possible to create components from a function, like so:

```js
function conditionalComponent(predicate) {
  if (predicate) {
    return <template><p>Cool</p></template>
  } else {
    return <template><p>Lame</p></template>
  }
}
```

However, doing so will have unexpected costs at runtime. It’s worth remembering what the resulting output is:

```js
function conditionalComponent(predicate) {
  if (predicate) {
    return setComponentTemplate(
      precompileTemplate(`<p>Cool</p>`),
      templateOnly()
    );
  } else {
    return setComponentTemplate(
      precompileTemplate(`<p>Lame</p>`),
      templateOnly()
    );
  }
}
```

The problem here is that this requires re-running both the creation of the template-only empty backing instance which has `null` for `this`, and the association between the two. As described above, the template precompilation also happens during the build, which eliminates some but not all of the apparent cost here; but the other parts are needlessly dynamic and expensive.

In this scenario, users can accomplish the same thing by manually hoisting the component definitions to module scope:

```js
const Cool = <template><p>Cool</p></template>;
const Lame = <template><p>Lame</p></template>;

function conditionalComponent(predicate) {
  if (predicate) {
    return Cool;
  } else {
    return Lame;
  }
}
```

(Notice that this is no different than any other concern around other costly operations needlessly happening in a function body repeatedly.)

See further discussion below under [**How We Teach This**](#how-we-teach-this).


### Interop

Since all existing components already work using the same low-level primitives the new system uses, strict-mode components using `<template>` can import and invoke components authored in Ember’s loose mode format. Similarly, since loose mode components are resolved to a component which is the default export of a module in the correct/conventional location on disk, components authored in strict mode with `<template>` and exported as the default export in that conventional location will be resolve-able by loose mode components as well.

There are two qualifications to the interop story: a minor one around named exports and a more significant one around non-colocated templates.


#### Named exports

However, other components exported as *named* exports will not be available in loose mode: resolution only evaluates default exports. This is a temporary incoherence which will be resolved as the ecosystem migrates to strict mode. As a workaround, users may choose to create reexport-only modules to allow loose-mode access. For example, given a module `app/components/named-only.js` with two named export components:

```js
export const Greet = <template>
  <p>Hello, {{@name}}!</p>
</template>

export const Farewell = <template>
  <p>Goodbye, {{@name}}!</p>
</template>
```

A user wishing to make these available to loose mode could introduce two new modules:

- `app/components/named-only/greet.js`:

    ```js
    export { Greet as default } from '../named-only';
    ```

- `app/components/named-only/farewell.js`:

    ```js
    export { Farewell as default } from '../named-only';
    ```

Then a loose-mode component could invoke them like `<NamedOnly::Greet @name="Chris">` and `<NamedOnly::Farewell @name="Krycho" />`. Note that this pattern is not at all *necessary* for migration, but may be useful.


#### Non-colocated templates

Since the same time as the Ember Octane release, Ember has supported “colocated templates,” where the template file for a component can live next to it in the `app/components` or `addon/components` directory, instead of in the `app/templates/components` or `addon/templates/components` directory:

```
my-ember-app/
  app/
    components/
      example.js
      example.hbs
```

Although this appears to be merely a user-facing convenience, there is a real and important difference at the implementation level which currently prevents first-class component templates from using classic, non-colocated templates:

Colocated templates are merged into the sibling JavaScript module at build time and set as the template for the component using `setComponentTemplate`: the same primitive used by first-class component templates. This includes template-only components, for which Ember's build synthesizes a JavaScript module and uses `templateOnlyComponent()`—again, just as `<template>` does.

By contrast, classic/non-colocated templates are *not* merged into the associated JavaScript module (if any). They remain as their own distinct module at runtime. Those template modules can be looked up via the AMD module system or Ember's DI registry, and Ember connects them to components via the same system it always did before the introduction of the `setComponentTemplate` primitive (and the same way it continues to connect the route-controller-template triplet). Critically, this means that as of today, Ember does *not* connect non-colocated templates to the associated class (whether backing class or `templateOnlyComponent()`-generated) using `setComponentTemplate`. This means that the corresponding `getComponentTemplate()` lookup used when resolving those components does not work. First-class component templates which reference non-colocated component templates will build successfully, but *do not render anything for them*.

This is a fairly serious developer experience problem, because it fails *invisibly* (see [this demo repository][non-colo-failure] to see this failure mode in practice).

[non-colo-failure]: https://github.com/chriskrycho/fcct-interop-demo

We can address in one of two ways:

- Introduce support into Ember to associated non-colocated templates with their associated classes.
- Introduce debug output which informs users that they must first migrate the referenced component to use colocation.

Of these, the second option is preferable. It has significantly lower risk of introducing bugs in the framework along the way, because it only requires adding some debug alerting and does *not* require changing the semantics or implementation of long-standing Ember features. It is [straightforward to codemod][colo-codemod] to colocation.


### The “prelude”

While all values used in templates must be explicitly in scope, Ember[^glimmer-prelude] will provide some via a “prelude”.[^prelude] These are always in scope and do not need to be imported. See [RFC #0496: Handlebars Strict Mode: Keywords][rfc-0496-keyword] for a detailed list of keywords and imports.

[rfc-0496-keyword]: https://emberjs.github.io/rfcs/0496-handlebars-strict-mode.html#keywords

[^glimmer-prelude]: Glimmer.js may provide its own prelude. While long-term the two should likely align, this RFC simply takes the _status quo_ as a given

[^prelude]: “Prelude” is the conventional name for this functionality in programming language design. See e.g. the discussion from [Rust’s `std::prelude` docs](https://doc.rust-lang.org/std/prelude/index.html).


### Tooling

To support the new format, we need to update tooling across the ecosystem to understand the format.


#### Syntax highlighting

First, we need syntax highlighting support across the ecosystem. Support already exists [for VS Code][vsc-highlighting], which represents the single largest group of web developers; as well as for any tool which can use [a tree-sitter grammar][tree-sitter-highlighting] (e.g. Neovim).

We will also need to implement support in [Linguist](https://github.com/github/linguist) for GitHub syntax highlighting.

Beyond that, we should encourage the community to add support for other editors (IntelliJ, Atom, Emacs, etc.) as well as for tools like [Rouge](https://rubygems.org/gems/rouge) (which powers GitLab syntax highlighting) and other highlighters, but need not treat those as *blocking* adoption of first-class component templates.

[vsc-highlighting]: https://marketplace.visualstudio.com/items?itemName=chiragpat.vscode-glimmer
[tree-sitter-highlighting]: https://github.com/alexlafroscia/tree-sitter-glimmer


#### Blueprints

Component and component test blueprints will need to be updated to support generating the new format. (See a forthcoming RFC for updates to testing to support this more robustly.) During the transition period, we should allow generating both. The rollout will follow the example of the rollout of Glimmer components with Octane:

1. Introduce the ability to author components in the new format with a new `--strict` flag, but leave the default today’s loose mode format. Introduce `--loose` as an explicit flag for using today’s loose mode format.

2. When Ember Polaris[^polaris] is released, change the default to the new format for apps and addons which set `"edition": "polaris"`, while leaving loose mode available via `--loose`, and preserving `--strict` as an explicit flag for the new default.

3. If or when loose mode templates are deprecated, the supporting blueprint infrastructure can be removed, including the `--loose` flag.

The current blueprints support generating a backing class for any existing component template which does not already have a backing class with the `component-class` format. We have two choices about the behavior of that blueprint for strict mode templates:

1. Do not support it. Adding a backing class is simply a matter of adding an import and adding a class.

2. Re-implement the blueprint using an AST transform (which we have prior art for: route generation uses that approach), to add a backing class for an existing default export in the module.

(We should do (1). The community can of course implement (2) if interested.)

We should also update the name of the class generated for a component class. The default behavior of today's blueprint when generating a component is to suffix the class name with `Component`. Thus, running `ember generate component greeting --component-class=@glimmer/component` will produce a class named `GreetingComponent`.[^ts-component-name]

There was room for debate about whether this made sense for naming component classes up till now, since the invocation name was based on the file name (using Ember's resolution rules) and *not* the class name. Now, though, it *will* be based on the imported name, and the standard behavior of auto-import tooling is to import classes by their full name—whether the item is a named export or a default export. When a user goes to auto-complete `Greeting` (e.g. in [Glint][glint]), they will end up with `GreetingComponent`, leading to this sort of thing if they don’t rename it:

```js
import GreetingComponent from './greeting';

<template>
  <GreetingComponent @name={{@user.name}} />
</template>
```

This is obviously undesirable, but avoiding this will mean mean renaming locally after the auto-complete works. That renaming operation is a needless paper cut in the best case of importing a default export. It rises to the level of a significant annoyance when using named imports:

```js
import {
  ButtonComponent as UIButton,
  FormComponent as UIForm,
  InputComponent as UIInput,
} from './ui';

<template>
  <UIForm @onSubmit={{@saveName}}>
    <UIInput @label="Name" @value={{@name}} @onUpdate={{@updateName}} />
    <UIButton @label="Save" type="submit" />
  </UIForm>
</template>
```

And it makes namespace-style imports basically unusable: to invoke *without* `Component` everywhere, you have to rebind all the imports you use!

```js
import * as _UI from './ui';
const UI = {
  Form: _UI.FormComponent,
  Button: _UI.ButtonComponent,
  Input: _UI.InputComponent
}

<template>
  <UI.Form @onSubmit={{@saveName}}>
    <UI.Input @label="Name" @value={{@name}} @onUpdate={{@updateName}} />
    <UI.Button @label="Save" type="submit" />
  </UI.Form>
</template>
```

Accordingly, we should switch to generating *without* a class name: `ember generate component greeting --component-class=@glimmer/component` should produce a class named `Greeting`, *not* `GreetingComponent`. The generated names for routes, services, and controllers can remain as they are, since they are never invoked this way.
  
[^polaris]: Polaris was announced as planned at EmberConf 2021. This plan assumes we ship Polaris before Ember 5. If we ship Ember 5 first, the dynamics would be much the same, but with the major version as the point when we switch the default instead.

[^ts-component-name]: In TypeScript, this also extends to `GreetingComponentArgs` (or, with [RFC #0748][rfc-0748], something like `GreetingComponentSignature`), which gets *really* unwieldy!


#### Linting and formatting

As with syntax highlighting, we need to support the new format with linting and formatting integration.

For linting, we need to make two changes:

1. Create an ESLint processor which uses the same Babel transform as the core behavior itself (as provided currently by [ember-template-imports][eti]) to make normal ESLint rules (e.g. unused imports and values) work with the template scope.

2. Going the other direction, make it possible to use the existing `ember-template-lint` rules in `.gjs`/`.gts` files. This likely means integrating `ember-template-lint` directly into ESLint, in much the same way that other sub-language integration is done (in the same way that e.g. [eslint-plugin-svelte3][eslint-svelte] integrates Svelte’s custom language).

[eslint-svelte]: https://github.com/sveltejs/eslint-plugin-svelte3

For formatting, we need to implement a custom parser plugin and language which will make Prettier able to format both the host JavaScript files and the embedded templates. This will need to present a view of the non-template parts of the file to Prettier so that it formats the JavaScript correctly without updating the template contents, and vice versa. The primary work here is to make it so that we can leverage Prettier’s existing support for JavaScript/TypeScript and Handlebars in a `.gjs`/`.gts` file (rather than simply ending up with a parse error, as happens when you try to treat those files as pure JS or TS).


#### Language server support

The final piece of tooling we need for supporting this is *language server support*. Language servers using the [Language Server Protocol][lsp] allow a variety of different editors (including e.g. Vim, Visual Studio Code, Emacs, Atom, Sublime Text, IntelliJ, and others) to use a single language server Currently, for uninteresting historical reasons, there are a handful of language servers floating around which Ember developers use. Most important for our purposes are the [Unstable Ember Language Server][uels] and [Glint][glint].

**Neither of these is technically a hard blocker for adopting first-class component templates, but we expect there to be significant community demand for support.** However, the existing support in these tools for the `hbs` experiment means that supporting `<template>` is relatively straightforward: the work needs to be done, but is not especially large. In particular, the same Babel transform which makes `<template>` work and can power ESLint and Prettier integration should provide the necessary information for language servers as well, which can then leverage their own interpretations of templates (e.g. Glint's mapping from a Handlebars template to a TypeScript representation) to provide richer feedback, auto-completion, go-to-definition, documentation hovers, etc.

[lsp]: https://microsoft.github.io/language-server-protocol/
[uels]: https://marketplace.visualstudio.com/items?itemName=lifeart.vscode-ember-unstable
[glint]: https://github.com/typed-ember/glint


#### Codemod

While providing a codemod is not a *hard* necessity, it is much like language server support: there will be high community demand.

Such a codemod will automate the fairly mechanical work of providing a wrapping `<template>` for template-only components and moving the content of an `.hbs` file into a `<template>` on the backing class for class-backed components. To do that, however, there are two major pieces such a codemod will need to address:

- **Identifying where a given component or item came from**. This is not *trivial*, since in most apps the items are all in one big global namespace. This is definitely *tractable*, though. A codemod could start by walking the graph of Ember addons any given library depends on and identifying all names it exports in terms of Ember's standard layout. Then that can be fed into each template module being converted.

    There will definitely be occasional conflicts here, for example when developers have *intentionally* overridden something supplied by an addon. In that case of conflict, the codemod can bail and report it to the end user. (We could use a telemetry-powered codemod like we did for the native classes codemod with Octane, but that's a much higher lift and my own judgment is that the cost-benefit ratio is low enough not to be worth it in *this* case. People generally either work around those or have done it on purpose.)

- **Handling non-colocated templates.** As discussed above in [**Detailed Design: Interop**](#interop), strict mode templates cannot *currently* resolve components where the template is still located in `templates/components` rather than next to a backing class, if any, in `components`. If we do not change this behavior at the framework level, we will need to recommend people start by migrating to colocated templates (which already has [a reliable codemod][colo-codemod]).

[colo-codemod]: https://github.com/ember-codemods/ember-component-template-colocation-migrator


### TypeScript

The type of a component is not affected by this proposal. However, it is worth seeing *how* a component defined using `<template>` works with types, at least for the purpose of documentation (and for integration with the current DefinitelyTyped definitions).

For a class-backed component, there is no change to the *types* of the component when using `<template>`. As described above in the discussion of language servers, tools like Glint will need to provide an interpretation of the body of a `<template>` which correctly understands the scope in which it is embedded, i.e. correctly providing `this` to it.

For a template-only component, defining the type will require a type import to represent that there is a component with no `this` context, etc. Glint already supplies such a type, albeit with the types updated for [RFC #0748][rfc-0748]. For today’s purposes, we can simply augment the [existing types on DefinitelyTyped][DT-toc] with `Args`.

[rfc-0748]: https://github.com/emberjs/rfcs/pull/748
[DT-toc]: https://github.com/DefinitelyTyped/DefinitelyTyped/blob/54d540ab4deb2588c0eff39dadf370cbf0a2dee4/types/ember__component/template-only.d.ts

<details><summary>updated signature on DT</summary>

```ts
declare const A: unique symbol;

// This class is not intended to be directly constructable.
declare class _TemplateOnlyComponent<Args extends {}> {
    // Type brand to simulate a nominal type.
    declare private brand: 'TemplateOnlyComponent';
    // Host to make args "used"
    declare private [A]: Args;
    toString(): string;
}

// Export an interface instead to prevent construction.
// tslint:disable-next-line:no-empty-interface
export interface TemplateOnlyComponent<Args extends {} = {}> extends _TemplateOnlyComponent<Args> {}
type TC<Args extends {} = {}> = TemplateOnlyComponent<Args>

declare function templateOnly(moduleName?: string): TemplateOnlyComponent;

export default templateOnly;

// Shut off automatic exporting.
export {};
```

</details>

Users can then define a named template-only component like this:

```ts
import type { TemplateOnlyComponent } from '@glimmer/component';

const Greet: TemplateOnlyComponent<{ name: string }> = <template>
  <p>Hello, {{@name}}!</p>
</template>
```

(While this empty interface is currently more or less useless from a type-checking perspective—we will need something like Glint to support it—it suffices to provide a hook for documentation tooling such as [TypeDoc][typedoc] or [API Extractor][api-extractor], and thus suffices for the level of support we have for TypeScript today.)

[typedoc]: https://typedoc.org
[api-extractor]: https://api-extractor.com

However, since the top-level `<template>` syntax is sugar for an anonymous default export, there is nowhere to put a type declaration like this. This is a limitation of default exports in JavaScript: functions and classes have names as part of their declarations, but other items do not, so they cannot be both named *and* part of the default export of a module.

Accordingly, we propose an extension to `<template>`, available only in `.gts` files, which uses the following syntax designed to mirror type parameterization in TypeScript but in a way that is straightforward to parse into the desired target format:

```ts
export interface GreetingArgs {
  name: string;
}

<template[GreetingArgs]>
  <p>Hello, {{@name}}!</p>
</template>
```

This syntax for generics has prior art in other programming languages, including Scala, Go, and Ruby’s Sorbet type checker (a cousin of TypeScript, as it were!). It clearly associates the `Args` with the `template`, while *not* putting it in a value space which could conflict with future extensions to `<template>` with “attributes” in the value space.[^emblem-etc]

Given this design, we can also simplify the definition of named components (both forms will of course be legal):

```ts
export interface GreetingArgs {
  name: string;
}

export const Greeting = <template[GreetingArgs]>
  <p>Hello, {{@name}}!</p>
</template>
```

There are two key restrictions here:

1. As mentioned above, this is illegal in the context of a class-backed component, because the component class itself is the host for the signature (and must be to make `this.args` type check correctly).

2. The only thing allowed within the `[...]` is a type available in the local scope. *It is not legal to provide an inline type definition.* However, given the relative verbosity of even today’s component signature, still less the revised version from [RFC #0748][rfc-0748], inline signatures are unlikely to be attractive anyway.

From an implementation perspective, this requires our language parser to handle this variant of the tag, and for the transforms supplied for compilation to properly ignore this for build output but to supply it in an appropriate place for TypeScript-aware tools like Glint to be able to take advantage of it.


### Custom file extension

These tooling considerations together provide the motivation for a custom file extension (`.gjs` and `.gts`). In the case of TypeScript in particular, it is not possible to *remove* errors using a TypeScript language server plugin, which means that in a pure `.js` or `.ts` file, a user would get conflicting reports from TypeScript and (e.g.) Glint. Thus, today, Glint recommends that GlimmerX users *disable TypeScript in their projects*, and rely on only Glint. Taking a lesson from Vue and Svelte, however, introducing a custom file extension allows us to provide a default type for `.gjs`/`.gts` files which makes TypeScript happy in `.js` and `.ts` files, and on top of which tools like Glint can safely add *more* information.[^hbs-custom-syntax]

While both Prettier and ESLint *can* work with `.js` or `.ts`, introducing the new file extension also simplifies the tooling implementation for them. It does mean that tools like GitHub’s Linguist will not work without implementing support, but we need to do that work anyway.

[^hbs-custom-syntax]: Note that this also applies to the `hbs` syntax discussed in [**Alternatives: Template literals (`hbs`)**](#template-literals-hbs).


## Transition path

We will complete this transition as part of Ember Polaris. To do that successfully, we must:

- implement the features as described in [**Detailed design**](#detailed-design), migrating in the implementation from `ember-template-imports`
- update the [tooling](#tooling):
  - [syntax highlighting](#syntax-highlighting)
  - [blueprints](#blueprints)
  - [linting and formatting](#linting-and-formatting)
  - [language servers](#language-server-support)
- update all [the teaching materials](#how-we-teach-this)

Additionally, an optimal transition will include changes to [language server implementations](#language-server-support) and supply a [codemod](#codemod) from loose to strict mode. (We may be *able* to release this as part of Polaris without those, but the transition will be much more successful with them.)

Finally, the Glimmer.js (and thus [GlimmerX][glimmerx]) implementation should update to match this, further decreasing the delta between standalone Glimmer and Ember.


## How we teach this

We describe a `<template>` as representing the *template* for a *component*. When there is no backing class, that’s all there is to the component. When there *is* a backing class, the component also has associated state and behavior. (Notably, this shift already began with Octane, where we generate template-only components by default.)

This explanatory model provides a helpful opportunity, when first introducing the idea of a backing class, to link to deeper-dive materials which let people who want to understand more deeply. In particular, it allows us a place to point out that mechanically, there *is* in fact always an associated component instance (generated via `templateOnlyComponent()`), but it’s just a way for this `<template>` to be hooked into the broader system, rather than a home for state.

**Note:** This is complicated by the need to introduce route templates as well as component templates. The text here assumes that we will, in parallel with this work, resolve the design questions addressed by [RFC #0731][rfc-0731] in something like the design proposed there. Accordingly, it also assumes we will update the generators and the relevant text accordingly. Fundamentally, we *should not* update the guides until we have a resolution for that design space, so they can be updated in a coherent way.


### Guides

Once `<template>` is implemented and tooling is sufficiently stable, we will update the guides with changes along the following lines:


#### Tutorial

One cross-cutting change here will be updating the output from generators, including correct new file extensions. The following discussion of the current sections of the guide *assumes* that change, and addresses concrete *pedagogical* changes we need to make. If a section is not included here, it (a) needs little or no other change beyond the minimum or (b) is dependent on the results of [RFC #0731][rfc-0731] to flesh out the details.


##### Orientation

Here we will need to update the prose to describe that `<template>` marks this as Ember/Glimmer’s special superset of HTML, with prose long the lines of:

> Note that all you need to do to have a working Ember component is to wrap your HTML in `<template>`.


##### Component Basics

Introducing components will see a lot of changes, unsurprisingly:

1. The introduction to components in updated guides will depend on the specific design choices we make in [RFC #0731][rfc-0731]. One possible approach here will be to note that we have already seen components in practice, if the decision in that space is that routes simply invoke a component template. Otherwise we may indicated that components are similar to route templates, but are self-contained.

2. The use of `<CapitalizedComponents />` is no longer required, but remains a *helpful convention*.[^resolver-capitalized-components] If someone does `const foo = <template>...</template>`, they will be able to invoke that as `<foo />` elsewhere. The notes in this section as well as about `LinkTo` will need to be updated to describe it accordingly.

3. After introducing `<Jumbo />` with updated use of a wrapping `<template>`, discuss *importing it* into the route (component) template which uses it. This is a good place to describe how `.gjs` can use JS features, and hint that we’ll see more of this later; it is also a good opportunity to note that we *could* have defined `<Jumbo />` locally, but that we moved it to a separate file because we’re sharing it across multiple different components.

4. Our discussion of the testing will need to be updated to include importing the components under test, and to use `<template>` rather than `hbs` strings for the `render` calls.[^testing-rfc]

[^resolver-capitalized-components]: Historically users *had* to use this convention, but only because that was the decision for how the resolver would work.


##### More About Components

We can simply *remove* the discussion of namespaced components, in favor of simply describing the use of normal JS imports to accomplish the same goal. However, here we can also note that JavaScript modules are a great way to organize groups of related components, and show how we might use namespace-style imports (`import * as Rental from '';` and then `<Rental.Image />` within a `<template>`) for this kind of organization.[^namespace-deprecation]

[^namespace-deprecation]: Attentive readers will likely have noticed that this makes the namespace sigil a candidate for later deprecation, since it will be entirely redundant once the ecosystem moves fully to strict mode and template imports. However, that question is best left to a potential future RFC deprecating loose mode.


##### Interactive Components

The discussion of adding behavior to components will need to be updated to account for the design *and* the new possibilities in the space:

1. Show that when we generate a class for an existing component, it  adds the `Component` import, creates a wrapping class, and moves the template into the body of that class. Here, teach the mental model that a `<template>` which is part of a class body has access to the instance properties on the backing class.

2. When discussing use of values from `ENV`, instead of providing a getter on a backing class, start by creating a `TOKEN` constant in module scope, and show that it is available to access in the template. In the following section, which shows args being used in the template, simply use that `TOKEN` value in the template directly, `access_token={{TOKEN}}`.


##### Reusable Components

This section provides us an opportunity to show how useful it can be to introduce local functions. The code samples here currently use a backing class, but they only do so to provide a home for getters which provide an encoded URI for the Mapbox token and derive the `src` from the arguments.

(I will provide this example in full here in part because it shows powerfully the pedagogical value of this RFC!)

The Mapbox token value is not reactive and therefore the computation has no reason to exist on a backing class *at all*. It is only there today because *without* first-class component templates, it requires introducing the heavier notion of classic (pre-[RFC #0756][rfc-0756]) helpers. With `<template>`, it can simply become a constant value defined in local scope:

```js
import ENV from 'super-rentals/config/environment';

const TOKEN = encodeURIComponent(ENV.MAPBOX_ACCESS_TOKEN)

<template>
  <div class="map">
    <img
      alt="Map image at coordinates {{@lat}},{{@lng}}"
      ...attributes
      src="https://api.mapbox.com/styles/v1/mapbox/streets-v11/static/{{@lng}},{{@lat}},{{@zoom}}/{{@width}}x{{@height}}@2x?access_token={{TOKEN}}"
      width={{@width}} height={{@height}}
    >
  </div>
</template>
```

Similarly, while `src` *is* computed from reactive data, there is once again no reason to compute it *in a class* if we are not going to use the class to store state. We can just write a local function and use it (and use that to note that we handle named arguments, too):

```js
import ENV from 'super-rentals/config/environment';

const TOKEN = encodeURIComponent(ENV.MAPBOX_ACCESS_TOKEN)
const MAPBOX_API = 'https://api.mapbox.com/styles/v1/mapbox/streets-v11/static';

function source({ lng, lat, width, height, zoom }) {
  let coordinates = `${lng},${lat},${zoom}`;
  let dimensions = `${width}x${height}`;
  let accessToken = `access_token=${TOKEN}`;

  return `${MAPBOX_API}/${coordinates}/${dimensions}@2x?${accessToken}`;
}

<template>
  <div class="map">
    <img
      alt="Map image at coordinates {{@lat}},{{@lng}}"
      ...attributes
      src={{source lng=@lng lat=@lat width=@width zoom=@zoom}}
      width={{@width}} height={{@height}}
    >
  </div>
</template>
```

Notice the results of this pedagogically:

- We have asked people to write *less code*: fewer imports, and less overall syntax.
- The code they *do* write feels much more HTML-first. The template can stay almost exactly as it was, with no shift to a backing class.
- For developers who have experience with other frameworks, this feels familiar, but with an Ember twist.

At this point we could *additionally* show that we could introduce a backing class, and discuss the trade-offs of introducing a class when we don't have any other local state. This also allows us to encourage just using functions unless you *do* need local state.

The section “Getting JavaScript Values into the Test Context” will also be possible to simplify: we will simply be able to introduce tracked state locally and update it directly, without special testing helpers. That will dramatically reduce the number of bespoke ideas we have to cover here. Much of the related work will be addressed in other RFCs, but being able to use the *same* primitives to bring values into scope for tests as we do in apps (immediately above!) will be very helpful in reducing what we have to cover in this section.


#### Core Concepts: Components

This entire section will also need to be substantially reworked. Once again, I am here summarizing the changes rather than trying to rewrite the guide in place. Each section represents a page to be changed; if a section is not mentioned, it needs no substantive changes—likely only switching over to using the `<template>` wrapper.

At some point in the course of this discussion, we should call out (e.g. with a “Zoey says” block) that users should treat `<template>` the same way they treat a costly function which produces a result for the life of the whole app, and should therefore avoiding using `<template>` in function bodies rather than hoisting them, etc. This cannot be a hard and fast rule about where `<template>` definitions live, because there are plenty of ways to do it safely, and what’s more we *need* to do it in test modules. The point is simply to align people’s mental model for `<template>` with *other* costly operations, since these concerns are not specific to component creation.


##### Introducing Components

Unsurprisingly, this is the section which will see the most sweeping changes.

- As described in the tutorial, our introduction will depend on the design chosen for route templates. We will either note that we’ve already seen our first component, if the application template *was* a component, or note the similarities and differences between route templates and components otherwise.

- We can continue to show breaking the component out into separate files, with a top-level `<template>` (serving, so far implicitly, as a default export).

- Then, back in the application file, we can show using `import` to refer to it.

- As in the tutorial, the discussion around naming will need to be updated to indicate that we capitalize by *convention*: it will no longer be a hard requirement. Likewise, the “Zoey says...” will go away because we will no longer be using resolution to get imports.

- After showing the other extraction-style refactors, we can show how components which don't need to be exported can just be defined locally with a `const` declaration, and explain that the standalone `<template>` tag is sugar for a default export. This will also provide the first hook for defining helpers etc. locally in following sections.

- We will entirely drop the folder namespace syntax (`::`) discussion, in favor of showing how normal JS imports handle that concern—including showing how the combination of named exports and namespace-style imports handle those. (This will necessitate reworking the example, which currently uses that namespacing as a means of scoping.)


##### Helper functions

Instead of introducing `app/helpers` and the resolution-based lookup, we can introduce the helper as a local function in the component which needs it. This will be the first place where this guide explicitly calls out that components have access to values in their surrounding scope, just like normal JavaScript. This will be a good point to call out the power and versatility this affords.

Additionally, instead of the *next* section being the place where we first identify that JS is needed to make our UI dynamic, we will address that here. The next section can then build on that by showing how classes make certain patterns *easier*.


##### Component State and Actions

Here, the content will need to shift in two ways:

1. The motivation for introducing a backing class shifts slightly: we have the ability to have state at the module level already, including via class-backed helpers. What we need is a way to have state that is for *each component instance*. A class is JavaScript’s first-class way of doing that, so we have a version of first-class component templates which supports it!

2. Having made the motivation clear, we can show the `<template>` in the body of the class and explain that it is exactly the same as a standalone template component, except that it now has access to the backing class for local state, "actions", etc.


##### Template Lifecycle, DOM, and Modifiers

Once again, many of the changes here will be mechanical: just using the new syntax. However, this also provides another opportunity to discuss (and demonstrate) the value of local-only vs. exported functionality. Both of the main custom modifier examples here currently show highly-reusable examples of modifiers which *should* be exported and should indeed probably live in their own modules. Accordingly, we might find an example which shows the value of having a locally-scoped modifier—e.g. something which manages the private details of an `iframe`.


### API Docs

There is presently no API for `<template>` itself, as proposed in this RFC, though it leaves room for future RFCs to do so.[^emblem-etc] Since there is no import location, we should cover it under the `@glimmer/component` module documentation. This will be a natural home for it, since we will always discuss it in the context of components.

[^emblem-etc]: Historically, for example, many Ember apps used [Emblem][emblem] as a templating language—and it is still possible to do so today! In the future, that could be supported with `<template lang="emblem">`. This would also be an easy home for experiments with a Svelte-like syntax with e.g. `<template lang="svelte">` etc.

[emblem]: http://emblemjs.com


### Existing Ember users

The Ember community has long experience with the idea that we can only have *one* component per file, and that component templates and the backing class must always be in separate files. The name of this feature, *first-class component templates*, is designed to help explain how it relates to that historical experience. In the past, templates in many ways were second-class citizens of the overall experience of authoring an Ember app—especially for template-only components. Adding even a small amount of functionality to a template came with a lot of friction and other downsides: switching to a class-backed component, or introducing a globally-available helper. Now, adding a local helper utility is no different for a template-only component than it is anywhere else: write a function!

As a result, we can teach this to existing Ember users as taking everything they already know about how templates work, while making it faster, easier, and lighter-weight to solve common problems. The one significant new concept here is the `<template>` tag. We can describe that to existing features as the one addition which raises templates up to being a first-class tool in your app or addon.

A blog post can introduce the feature along these lines when the feature ships, with examples showing how it simplifies existing patterns and enables new capabilities. For simplifying existing patterns, we might pull the same Mapbox example from the guides shown above. For enabling new capabilities, we could show how it enables using native CSS Modules with none of the hacks required by Ember CSS Modules.


## Drawbacks

- Since there is no notional import for `<template>`, there also isn’t a notional home for API documentation for it other than components.

- We must build a custom tooling integration with Prettier for the file format to *parse*. (As discussed below, we must build custom tooling to *use* Prettier for other options, but Prettier can *parse* them without custom tooling.)

- Developers may put an unreasonable number of components, helpers, modifiers, etc. in a single file, degrading the maintainability of that module. However, the counterpoint here is that large files are already common in many code bases, with or without this tool. Indeed, that happens in non-UI and UI code bases alike!

    Moreover, experience from frameworks which restrict component authoring formats to a single component per file, including Ember’s loose mode templates as well as Vue and Svelte SFCs, is that those components themselves tend to balloon in size. Sometimes that’s because everything in those components is notionally related or because much of it should be treated as "private API" for that component (even if it would be helpful to refactor small local components). Sometimes it is just because of the annoying overhead of needing to create a separate file to break the huge component into smaller pieces, and then import them all (or make them globally available, in Ember loose mode template!).

    The analogy here would be if a JavaScript module could only have a single function or class in it, or a CSS file could only have a single declaration in it, regardless of what actually made sense for that particular module.

- The syntax offered here, `<template>`, overlaps with [a platform built-in][platform-template], and would look very strange if a user *did*  want to use the built-in form. This may provoke some degree of confusion for users if they are familiar with it. However, there are several reasons to think this drawback is not significant:

    - In practice `<template>` is very-little used, and only in the context of progressive enhancement with vanilla JS—not with frameworks.

    - Although it looks a little odd, the platform-native `<template>` can still be nested *within* a `<template>` tag as defined here.

    - Other frameworks (most notably Vue) have used `<template>` in much the same way we are here with no major confusion on the part of developers.

    - Most importantly, there is no *actual* conflict with the platform built-in, since `<template>` is not *JavaScript* syntax, which is where we are using it.

    As a bonus: in a certain sense, the use of `<template>` here “rhymes” with the version from the platform: it represents the dynamic HTML content associated with some JavaScript functionality.

- Some developers prefer to keep a hard file-level separation between JavaScript and HTML. This proposal allows that to continue for loose mode components, but not for strict mode components, and strongly suggests a future where it is *not* possible (if we deprecate loose mode in the future).

[platform-template]: http://developer.mozilla.org/en-US/docs/Web/HTML/Element/template


## Alternatives

Within the major strokes of this design proposal, we could tweak the invocation for the template space to clarify that it does *not* overlap with the built-in `<template>` tag.

- Use `<Template>` or `<Glimmer>` or similar. This would disambiguate it from the built-in `<template>`, but would introduce ambiguity with component invocation.
- Use a new sigil—much as we use `<:main>...</:main>` for named blocks, we could do `<[template]>...</[template]>` or something similar. While verbose and not especially pretty, this avoids overloading the platform tag.

There are also alternative possibilities for defining the type of a non-class-backed `<template>`, for the choice of consistency of `<template>` between class-backed and non-class-backed components, and for the syntax for *some* sort of strict mode templates.


### TypeScript signature

Instead of adding the generic position to `<template>`, we can simply recommend that TypeScript users always create a named `<template>` with a `const` binding, and then `export default` that named export:

```js
import type { TC } from '@glimmer/component';

const Greet: TC<{ name: string }> = <template>
  <p>Hello, {{@name}}!</p>
</template>

export default Greet;
```

(Users may also be tempted use an `as` cast after the `<template>`—but this is unsafe: it allows users to unsafely provide a narrower type than the item actually provides, whereas assignment only allows *widening* of the types.)

This works, and it simplifies the burden of the tooling implementation, but it comes with the significant downside of making a *much* worse authoring experience for TypeScript users than for JavaScript users.


### Distinguishing class-backed and template-only components

There is a small pedagogical difficulty, suggested by some of the language above, about the fact that we use `<template>` here to represent both the entirety of a component, when it is free-standing; and also the template portion of a component, when it is embedded in a class. Similarly, the proposed syntax for a TypeScript type signature must forbid the type parameter in class-backed components, because the correct home for the 

We could instead introduce `<component>` and `<template>` as separate constructs, where `<template>` provides a template definition for the class it is embedded in, and `<component>` defines a standalone component. In this approach, `<component>` could *not* be used within the body of a class, nor `<template>` in a standalone form.[^route-template]

The major downside here is that the transformation of adding a backing class becomes a bit more involved: not just moving a `<template>` definition into the new class body, but moving a `<component>` into the new class body and then changing it from `<component>` to `<template>`. Notice, however, that the move for TS users *already* involves some further transformation, even if we chose not to ship the `<template[Signature]>` form, because type parameters have to move. The same goes for any documentation attached to a `<template>` declaration when moved to a backing class: it has to go on the class itself instead.

**This is a reasonable alternative and *we should strongly consider adopting it*.** I did not propose it here because I think just using `<template>` is more or less comparable to having both `<template>` and `<component>` on balance, and having *only* `<template>` feels a little nicer. That is, however, a purely subjective judgment and I would be perfectly happy with a solution using both `<component>` and `<template>`.

[^route-template]: One other *possible* upside is that we could then in theory use `<template>` in the context of routes—but it is not clear that that is preferable to the direction suggested by [RFC #0731][rfc-0731]. My own judgment is that using `<template>` that way would be a mistake.


### Alternative syntaxes

Additionally, there are three major alternatives which Ember community members have proposed in the design space:

- **imports-only:** a design which uses “front-matter” to add imports, and only imports, to templates, while maintaining everything else in today’s system
- **single-file components (SFCs)**: a design which follows the example of Svelte and Vue and make HTML the basis of a component, and use a `<script>` tag to host JavaScript functionality
- **`hbs` template literals**: a design which mirrors the `<template>` design quite closely, but uses `hbs` template literals similar to those we use in tests today

I discuss each of these briefly below; for a *much* longer and more thorough discussion, please see the \~16,000-word series of blog posts I wrote as a deep dive: [**Ember Template Imports**](https://v5.chriskrycho.com/journal/ember-template-imports/). Notably, as I alluded to above, *all* of them require custom parsing implementation for tooling, especially including Prettier and language servers.


#### Imports-only

The imports-only design borrows the idea of “front-matter” from many text authoring formats, using something like `---`-delimiters to introduce a new, non-Handlebars area at the top of a template which allows exactly and only *imports* to appear. As with all strict-mode designs, all non-built-in values must be imported here. Thus, the *template* for the final component shown in the motivating example might appear like this:

<details><summary>motivating example shown with imports-only</summary>

- `greet.hbs`:

    ```hbs
    <p>Hello, {{@name}}!</p>
    ```

- `set-username.js`:

    ```js
    import Component from '@glimmer/component';
    import { tracked } from '@glimmer/tracking';

    export default class SetUsername extends Component {
      @tracked name = '';

      updateName = ({ target: { value } }) => {
        this.name = value;
      }

      saveName = (submitEvent) => {
        submitEvent.preventDefault();
        this.args.onSaveName(this.name);
      };
    }
    ```

- `set-username.hbs`:

    ```hbs
    <form {{on "submit" this.saveName}}>
      <label for='name'>Set username:</label>
      <input
        id='name'
        value={{this.value}}
        {{on "input" this.updateName}}
      />
      <button type='submit' disabled={{eq this.value.length 0}}>
        Generate
      </button>
    </form>
    ```

- `replace-location.js`:

    ```js
    export default function replaceLocation(el, { with: newUrl }) {
        el.contentWindow.location.replace(newUrl);
    });
    ```

- `generate-avatar.js`:

    ```js
    import Component from '@glimmer/component';
    import { tracked } from '@glimmer/tracking';
    import Greet from './greet.glimmer';
    import SetUsername from './set-username.glimmer';

    export default class GenerateAvatar extends Component {
      @tracked name = "";

      get previewUrl() {
        return `http://www.example.com/avatars/${name}`;
      }

      updateName = (newName) => {
        this.name = newName;
      };
    }
    ```

- `generate-avatar.hbs`:

    ```hbs
    ---
    import Greet from './greet';
    import SetUsername from './set-username';
    import replaceLocation from '../modifiers/replace-location';
    ---

    <Greet @name={{this.name}} />
    <SetUsername
      @name={{this.name}}
      @onSaveName={{this.updateName}}
    />

    {{#if (gt 0 this.name.length)}}
      <iframe
        title='Preview'
        {{replaceLocation with=this.previewUrl}}
      >
    {{/if}}
    ```

</details>

The major upside to this is that it is the smallest possible delta over today’s implementation. It also allows users who appreciate the separation between JavaScript and template files to maintain that. However, it has a number of significant downsides which render it much worse than the first-class component templates proposal, and in some cases worse than the _status quo_.

First, as with today’s _status quo_, it does not allow locally-scoped JavaScript values (including helpers and modifiers but also ecosystem tooling like GraphQL values, CSS-in-JS tooling, etc.) even when that is a perfectly reasonable design decision.

Second, it substantially complicates the implementation of tooling for language servers, which have to do extra work to detect the presence of a backing class and “stitch together” the backing class and the template if a backing class does exist.

Third, Since there are separate files for a template and its backing class, users may be tempted try to implement JavaScript functionality in the module for the backing class, and import it in the template:

```js
import Component from '@glimmer/component';

export function isBirthday() {/*...*/}

export default class MyComponent extends Component { /*...*/ }
```

```hbs
---
import { isBirthday } from './my-component';
---
{{#if (isBirthday @user.name)}}
  <p>Happy birthday, {{@user.name}}!</p>
{{/if}}
```

While a colocated template (the default since Octane) is part of the same module as the backing class, this does technically work![^recursive-module] However, it’s the kind of extremely surprising and weird thing we would generally try to avoid pedagogically—it requires us to explain that these two separate files (`.js` and `.hbs`) are combined into a single module at build time… and that we have nonetheless kept them separate at authoring time, requiring these kinds of workarounds.[^recursive-import-perf]

Perhaps most critically, this is **much worse than the _status quo_ for tests**.

If we support strict mode for tests, then out of the box we require people’s test authoring format to become *massively* more verbose and less useful, with imports in every single test `hbs` string, to support strict mode for tests:

```js
import { module, test } from 'qunit';
import { hbs } from 'ember-cli-htmlbars';
import { setupRenderingTest } from 'ember-qunit';
import { render } from '@ember/test-helpers';

module('demonstrates the problem', function (hooks) {
  setupRenderingTest(hooks);

  test('by rendering an imported component', async function (assert) {
    await render(hbs`
      ---
      import ComponentToTest from 'my-app/components/component-to-test';
      ---

      <ComponentToTest />
    `);
  });

  test('then again with an argument', async function (assert) {
    await render(hbs`
      ---
      import ComponentToTest from 'my-app/components/component-to-test';
      ---

      <ComponentToTest @anArg={{123}} />
    `);
  });
});
```

There are two major problems to notice here:

1. There is no way to import `ComponentToTest` here just once. This overhead will multiply across the number of items to reference in a given test—every component, every helper, every modifier!—as well as across the number of tests. This is a *large* increase in the burden of testing compared to today.

2. This also requires us to maintain, *and to teach*, the `hbs` handling for tests (or to design some replacement for it), on top of the “regular” template handling for components. This is the same situation as in Ember apps today—but since first-class component templates allow us to *improve* the consistency between app code and test code, this counts as a negative by comparison!

To get around this, we could continue to support a completely separate design for testing than for app code. In that case, though, if we want to support strict mode templates in tests, we need a separate authoring format for tests from app code. In fact, it basically requires that we fully implement something like the first-class component templates design!

[^recursive-module]: To see this for yourself, follow the instructions in [this gist](https://gist.github.com/chriskrycho/dc5adabd80c04d405c7a4894c0ffb99e). I had to test this out myself, and while it’s actually *very good* that modules work this way, I was initially surprised by it! If you’re curious: imports and exports are *static* and so are analyzed before the module is executed.

[^recursive-import-perf]: Without additional post-processing, this would also introduce extra runtime overhead in terms of the imports!


#### SFCs

Single File Components (hereafter SFCs) start with an HTML baseline and layer on functionality in a `<script>` tag, modeled on HTML’s own design, but with extra semantics supporting imports and making an `export default class extends Component` statement provide the `this` for the template context. It can, however, define modifiers and helpers local to the component. You can think of this as a fairly natural (and HTML-like) extension to the imports-only design.

<details><summary>motivating example shown with SFCs</summary>

- `greet.glimmer`:

    ```hbs
    <p>Hello, {{@name}}!</p>
    ```

- `set-username.glimmer`:

    ```hbs
    <script>
      import Component from '@glimmer/component';
      import { tracked } from '@glimmer/tracking';

      export default class SetUsername extends Component {
        @tracked name = '';

        updateName = ({ target: { value } }) => {
          this.name = value;
        }

        saveName = (submitEvent) => {
          submitEvent.preventDefault();
          this.args.onSaveName(this.name);
        };
      }
    </script>

    <form {{on "submit" this.saveName}}>
      <label for='name'>Set username:</label>
      <input
        id='name'
        value={{this.value}}
        {{on "input" this.updateName}}
      />
      <button type='submit' disabled={{eq this.value.length 0}}>
        Generate
      </button>
    </form>
    ```

- `generate-avatar.glimmer`:

    ```hbs
    <script>
      import Component from '@glimmer/component';
      import { tracked } from '@glimmer/tracking';
      import Greet from './greet.glimmer';
      import SetUsername from './set-username.glimmer';

      function replaceLocation(el, { with: newUrl }) {
        el.contentWindow.location.replace(newUrl);
      }

      export default class GenerateAvatar extends Component {
        @tracked name = "";

        get previewUrl() {
          return `http://www.example.com/avatars/${name}`;
        }

        updateName = (newName) => {
          this.name = newName;
        };
      }
    </script>

    <Greet @name={{this.name}} />
    <SetUsername
      @name={{this.name}}
      @onSaveName={{this.updateName}}
    />

    {{#if (gt 0 this.name.length)}}
      <iframe
        title='Preview'
        {{replaceLocation with=this.previewUrl}}
      >
    {{/if}}
    ```

</details>

This is very attractive in some ways, but it comes with three downsides:

First, SFCs do not allow multiple components to be defined in a single file. This has been an ongoing sticking point with the Vue and Svelte designs, such that there is even an [RFC for Svelte for supporting at least a subset of this functionality](https://github.com/sveltejs/rfcs/blob/inline-components/text/0000-inline-components.md).

Second, the scope handling for the default export is unusual and requires additional teaching.

Third, and again most critically: as with the imports-only design, **the SFC design requires that we continue to support a completely separate design for testing than for app code or use an incredibly verbose test authoring format.** If we maintain the same authoring format, we end up with the same problems as in the imports-only proposal:

```js
import { module, test } from 'qunit';
import { hbs } from 'ember-cli-htmlbars';
import { setupRenderingTest } from 'ember-qunit';
import { render } from '@ember/test-helpers';

module('demonstrates the problem', function (hooks) {
  setupRenderingTest(hooks);

  test('by rendering an imported component', async function (assert) {
    await render(hbs`
      <script>
        import ComponentToTest from 'my-app/components/component-to-test';
      </script>

      <ComponentToTest />
    `);
  });

  test('then again with an argument', async function (assert) {
    await render(hbs`
      <script>
        import ComponentToTest from 'my-app/components/component-to-test';
      </script>

      <ComponentToTest @anArg={{123}} />
    `);
  });
});
```

Besides having all the same problems as the imports-only approach does, notice that this also substantially increases the cost of tooling even at the level of syntax highlighting, because now we need multiple nested layers of syntax highlighting: `hbs` strings include both HTML and JavaScript! While some syntax highlighters support this, it is a much higher lift for those which do not.

And, once again, avoiding those problems more or less requires that we fully implement first-class component templates to avoid this!


#### Template literals (`hbs`)

The “template literals” design takes as its starting point the `hbs` template strings Ember has used for testing since the 1.x era. It is relatively similar to the `<template>` design, in that it uses JavaScript/TypeScript files as the basis for its design. Unlike `template`, it uses an explicit `hbs` import, presumably from `@glimmer/component`. For components with a backing class, the template is defined as a static class field.

<details><summary>Motivating example shown with <code>hbs</code></summary>

```js
import Component, { hbs } from '@glimmer/component';
import { tracked } from '@glimmer/tracking';

const Greet = hbs`
  <p>Hello, {{@name}}!</p>
`;

class SetUsername extends Component {
  @tracked name = '';

  updateName = ({ target: { value } }) => {
    this.name = value;
  }

  saveName = (submitEvent) => {
    submitEvent.preventDefault();
    this.args.onSaveName(this.name);
  };

  static template = hbs`
    <form {{on "submit" this.saveName}}>
      <label for='name'>Set username:</label>
      <input
        id='name'
        value={{this.value}}
        {{on "input" this.updateName}}
      />
      <button type='submit' disabled={{eq this.value.length 0}}>
        Generate
      </button>
    </form>
  `;
}

function replaceLocation(el, { with: newUrl }) {
  el.contentWindow.location.replace(newUrl);
}

export default class GenerateAvatar extends Component {
  @tracked name = "";

  get previewUrl() {
    return `http://www.example.com/avatars/${name}`;
  }

  updateName = (newName) => {
    this.name = newName;
  };

  static template = hbs`
    <Greet @name={{this.name}} />
    <SetUsername
      @name={{this.name}}
      @onSaveName={{this.updateName}}
    />

    {{#if (gt 0 this.name.length)}}
      <iframe
        title='Preview'
        {{replaceLocation with=this.previewUrl}}
      >
    {{/if}}
  `;
}
```

</details>

This has a few significant advantages!

First, unlike with the `<template>` proposal, Prettier can *parse* a file using `hbs` string with no further changes. (It cannot *format* them, however: it treats the contents of the string opaquely.) This is a small, but significant, decrease in the cost of supporting the format both up front and over time.

Second, it feels familiar to developers in the Ember ecosystem used to using `hbs` strings with their tests.

Third, the broader JavaScript ecosystem makes use of a number of template string syntaxes, e.g. with `css` from [Emotion][emotion] or [`graphql` from Apollo][graphql], so it has familiarity for people coming from *outside* the Ember ecosystem as well.

Fourth, we do not need to introduce custom syntax for providing types: we can type `hbs` as a function which accepts the args/signature as a type parameter, and teach people to perform.

Finally, it shares many of the other strongly-positive properties of the `<template>` design, including that it works exactly the same way for testing and app code.

These advantages are strong enough that this is *absolutely* the second-best move in the design space for us, and given a choice between maintaining the status quo and using `hbs` (i.e. if `<template>` were off the table), ***we should absolutely choose `hbs`***.

Those positives notwithstanding, **it also has some significant disadvantages compared to `<template>`**.

First, the design re-purposes JavaScript syntax and gives it totally different semantics—like `<template>` does with HTML, but with *zero* signal from the context that it is doing something special.

- `hbs` is *not* actually a template string literal; it is a compile-time macro. Attempting to use it like a template string literal (e.g. by using `${...}` string interpolation) is a build-time error. This substantially undercuts the familiarity of the design: `css` and `graphql` and similar are *actual* string templates, not compile-time macros, and accordingly developers can use normal JavaScript semantics with them.

- The use of `static template = ...` has the wrong semantics: static class fields do not have access to an instance’s `this`—but templates quite expressly *do*. The whole point of a component with a backing class is to provide a normal JavaScript `this`, so this is a significant mismatch, which has consequences for both teaching and tooling.

Second, the learning path is *much* less gradual: the simplest possible component requires showing and at least minimally explaining JavaScript import and export semantics and template literals *and* that it isn’t a normal template literal as described above.

Third, explaining what exactly `hbs` invocations produce is also strange: they aren’t actually JavaScript expressions, but they *appear* to be. In a template-only context, “invoking” `hbs` produces a component; in a class, it produces the *template* for that component. This is the same as with the `<template>` proposal, but it has the additional quirk of using JavaScript syntax to do it, rather than shifting languages.

Fourth, while supplying a type definition which allows `hbs` to receive a type parameter initially appears nicer than the custom syntax for `<template>`, that form *appears* like an unsafe type cast in TypeScript, the same as writing `as TC<Signature>` after the definition. In terms of how we implement the transform, it would actually be safe in practice (the compiled output would be the same as with `<template[Signature]>`, and therefore would constrain the body of the template in the same way)—but only because the thing passed to `hbs` is *not* a template string. People can therefore not rely on any of their intuitions on the TypeScript side, either.

Net, while there are some nice features to the `hbs` proposal, it comes out significantly worse than `<template>` in most ways we care about. The decreased tooling costs are real, but they are much smaller than the other downsides of the format.


## Unresolved questions

- Introducing a new file extension also provides an easy opportunity to change the default component manager for class-backed components in, and only in, the new file type—eliminating the need to subclass from Glimmer's `Component`. From the motivating example:

    ```js
    class SetUsername {
      @tracked name = '';

      updateName = ({ target: { value } }) => {
        this.name = value;
      }

      saveName = (submitEvent) => {
        submitEvent.preventDefault();
        this.args.onSaveName(this.name);
      };

      <template>
        <form {{on "submit" this.saveName}}>
          <label for='name'>Set username:</label>
          <input
            id='name'
            value={{this.value}}
            {{on "input" this.updateName}}
          />
          <button type='submit' disabled={{eq this.value.length 0}}>
            Generate
          </button>
        </form>
      </template>
    }
    ```

    However, this has unresolved complexities around providing the types needed for Glint, which requires a home for the information about the element(s) and yield(s) for the component. Today, Glint uses type-only declarations on Glimmer `Component`, which cannot be straightforwardly translated to this mode. This is likely tractable, and a future RFC may introduce it (including for defaulting `.gjs` and `.gts` into it automatically), but it is large enough that it is probably worth addressing separately.

- Does the possible confusion with the platform `<template>` warrant adopting an alternative syntax, whether component-like (`<Template>`) or using an additional sigil (`<[template]>`, `<% ... %>`, `<$ ... $>` etc.)? If so, what design? Here we must keep in mind that the design should not be ambiguous with “dynamic” behavior (e.g. `<{template}>` which is suggestive of the Svelte and React expression marker, and which we might find attractive for future iterations of template language ourselves).

- Are `.gjs` and `.gts` the best file extensions?

- How does this relate to the currently un-merged [RFC #0731: Add `setRouteComponent` API][rfc-0731]? That is: can we merge this and proceed with authoring components while there is an unresolved design problem for the related issue of routes, controllers, and their host components? Or should we see this as a helpful part of resolving *that* design question? (I believe we can move forward in parallel.)

    The primary challenge here is that, as things stand, our guides would have some fairly substantial incoherence until we solve the problems which #0731 is addressing: route templates would be totally different from component templates in their semantics and behavior. This points to the ongoing and increasing divergence of the `Route` and `Controller` design from the rest of the framework, but it’s directly connected to *this* RFC pedagogically.

- Should we include a plan for a staged rollout of deprecating namespace resolution in this RFC, rather than tackling it in a later RFC?

[rfc-0731]: https://github.com/emberjs/rfcs/pull/731
