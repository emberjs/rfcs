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


# First Class Component Templates


## Summary

Adopt `<template>` tags as a format for making component templates first-class participants in JavaScript and TypeScript with [strict mode][rfc-0496] template semantics. In support of the new syntax, adopt new custom JavaScript and TypeScript files with the extensions `.gjs` and `.gts` respectively.

First class component templates address a number of pain points in today’s component authoring world, and providing a number of new capabilities to Ember and Glimmer users:

- accessing local JavaScript values with no ceremony and no backing class, enabling much easier use of existing JavaScript ecosystem tools, including especially styling libraries—standard [CSS Modules][css-modules] will “just work,” for example

- authoring more than one component in a single file, where colocation makes sense—and thereby providing more control over a component’s public API

- likewise authoring locally-scoped helpers, modifiers, and other JavaScript functionality

First class component templates offer these new capabilities while not only maintaining but *improving* Ember’s long-standing commitment to integrated testing, in that it allows app and test code to share a single authoring paradigm—substantially simplifying our teaching story. Similarly, it preserves Ember’s long-standing commitment to treating JavaScript and HTML (and CSS!) as distinctive concerns which, however closely related, are not the *same*.

<details><summary>Full-fledged example showing how this might work in practice</summary>

(Note that for this and all following examples, I assume [RFC #0757: Default Modifier Manager][rfc-0757] for simplicity, but it does not meaningfully change *this* proposal.)

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
  - [Testing](#testing)
  - [The solution](#the-solution)
  - [Constraints](#constraints)
- [Detailed design](#detailed-design)
  - [Compilation](#compilation)
    - [Bound to a name](#bound-to-a-name)
    - [Standalone](#standalone)
    - [Class body](#class-body)
  - [Interop](#interop)
  - [The “prelude”](#the-prelude)
  - [Tooling](#tooling)
    - [Blueprints](#blueprints)
    - [Linting and formatting](#linting-and-formatting)
    - [Language server support](#language-server-support)
- [How we teach this](#how-we-teach-this)
  - [Guides](#guides)
    - [Tutorial](#tutorial)
    - [Core Concepts: Components](#core-concepts-components)
  - [API Docs](#api-docs)
  - [TypeScript](#typescript)
  - [Existing Ember users](#existing-ember-users)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Imports-only](#imports-only)
  - [SFCs](#sfcs)
  - [Template literals (`hbs`)](#template-literals-hbs)
    - [Advantages](#advantages)
    - [Disadvantages](#disadvantages)
- [Unresolved questions](#unresolved-questions)


## Motivation

Today, authors of Ember and Glimmer apps and libraries must author their templates and JavaScript in separate `.hbs` and `.js` files, and the templates exist in a “resolution” mode where every component, helper, and modifier exists in a single global namespace. This has a number of significant downsides. What’s more, there are significant new capabilities for Ember and Glimmer authors made available by embracing JavaScript scope—while keeping our commitments to testing and separation of concerns.[^original-primitives-rfc]

[^original-primitives-rfc]: See also [the SFC & Template Import Primitives RFC](https://github.com/emberjs/rfcs/blob/tomdale/template-primitives/text/000-sfc-and-template-import-primitives.md), which described the motivation for implementing the primitives on which this proposal will build.


### Namespaces and Modules

First, because of the global namespace, name conflicts are common, and avoiding them requires either manually namespacing components or using (now-deprecated) experimental tools like [`ember-holy-futuristic-template-namespacing-batman`](https://github.com/rwjblue/ember-holy-futuristic-template-namespacing-batman). But:

- Manually namespacing is clunky and does not actually *guarantee* there won't be conflicts. Combined with the way addons typically supply their components, helpers, and modifiers into the app namespace, name conflicts are sometimes unavoidable.

- Even the workaround via `ember-holy-futuristic-template-namespacing-batman` requires using different names for modules in Ember than their Node package name when the Node package uses [npm scopes][scopes]. (This was one of the original motivations for exploring a design which leverages JavaScript modules, in fact!) Since our resolution modes must ultimately deal in JavaScript terms, we are in the position of always potentially being one ecosystem shift away from another syntax conflict with template-language-only designs for managing scope.

- It requires our tooling to understand Ember's resolution rules, and cannot take advantage of existing ecosystem tooling. Our language servers, for example, have to more or less reimplement Ember’s resolver themselves. And on the build side, Ember CSS Modules has to do an enormous amount of very hacky work (just ask the author!) to make CSS Modules work with Ember. These problems are fundamental to the current model.

- There is a substantial performance cost to dynamically resolving components by name at runtime. This can be mitigated by using the combination of something like `ember-holy-futuristic-template-namespacing-batman` with the [strict resolver][strict-resolver], but the strict resolver is not standard—and really cannot be without something like this proposal.[^strict-resolver-rfc]

- It is extremely unpleasant (though, strictly speaking, *possible* as of Ember 3.25[^verbose-local]) to introduce a component, helper, or modifier which is not in the global namespace. See the next section, [**Scope**](#scope), for further on this.

The global namespace also comes with overhead for our teaching story by introducing a layer of “magic”: people just have to memorize that a file with a default export in a given location *automagically* is available with a given name. This is just a “bare fact”: there is nothing to connect it to in terms of a developers’ existing JavaScript or HTML knowledge.

[strict-resolver]: https://github.com/stefanpenner/ember-strict-resolver

These problems are all well-solved already, using the JavaScript modules spec (or "ESM", for ECMAScript Modules). Today, however Ember developers cannot take advantage of those or the tooling which understands them!

[^strict-resolver-rfc]: See the still-open [RFC #0683][rfc-0683] for a discussion of the full set of concerns involved in resolution, which include but are not limited to the template concerns addressed here.

[rfc-0683]: https://github.com/emberjs/rfcs/pull/683

[^verbose-local]: In all cases, doing so requires introducing a backing class to make the value available to the template *or* writing Ember's strict mode template syntax manually (which is error-prone and extremely verbose: it is designed as an output format, not an authoring format).


### Scope

Second, and closely related to the global namespace problem: there is presently no good way for users to introduce or use locally-scoped code. Every component, helper, and modifier must live in its own file, and be globally availbale—even if it is meant to be used privately. Where JavaScript modules provide users to control their public APIs in terms of `export`s, Ember apps largely cannot take advantage of those for anything which interacts with the template layer.

In practice, this has a number of knock-on effects for Ember code.

First, components tend to grow without bound, because the equivalent of the "extract method" or "extract into new class" refactorings (which we commonly use on the JS side) end up with two downsides:

- they make the newly-extracted components available to the whole app, even if the concern is private to that component
- the require an entirely new file, which is friction both for the creation and the use/understanding of a given view

Second, users also often introduce classes with actions or getters where a simple [function-based helper][rfc-0756] would do, because that is the only way to provide a non-global function. (I show this by example in [**How We Teaching This: Guides: Tutorial: Reusable Components**](#reusable-components) below.)

Third, it likewise incentivizes the use of the ember-render-modifiers with backing classes, rather than custom modifiers, because the behavior can then be scoped to that module—whereas, again, a custom modifier would be in global scope. This in turn makes it easy for users to miss the helpful separation of concerns which custom modifiers enable.

[rfc-0756]: https://github.com/emberjs/rfcs/pull/756

Over time, these all lead to a proliferation of backing classes which are only present to work around the fact that we have no *other* way to provide non-global scope for our components. These classes in turn tend to act as “state attractors,” leading to an unnecessary proliferation of state throughout an app or addon.

[scopes]: https://docs.npmjs.com/cli/v8/using-npm/scope

What’s more, tools which assume they will be used in JavaScript contexts more or less don’t work with our templates today, because the templates have no way to access them. Think of CSS tools like [CSS Modules][css-modules], which is widely used in the Ember ecosystem via [Ember CSS Modules][e-css-m]: our current implementation has to jump through many hoops and do many hacks to work at all. A format which makes JavaScript values available in template scope would let us drop *all* of that special sauce—and this goes for *all* such JavaScript-side tooling.

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

To address these problems, the Ember community [proposed primitives][rfc-0454] which unlocked experimentation in this space and [defined the semantics of “strict” templates which use those primitives][rfc-0496]. Now, with a history of having done that experimentation—with [GlimmerX][glimmerx] and [ember-template-imports][eti]—and having had *many* discussions about the tradeoffs over the years, it’s time to ship a proposal which resolves these questions: **first-class component templates**.

[rfc-0454]: https://github.com/emberjs/rfcs/pull/454
[rfc-0496]: https://emberjs.github.io/rfcs/0496-handlebars-strict-mode.html
[glimmerx]: https://glimmerjs.github.io/glimmer-experimental/
[eti]: https://github.com/ember-template-imports/ember-template-imports

In this new world, templates are authored in JavaScript files with a `<template>` tag. Templates defined this way have normal Glimmer template semantics, but instead of using a runtime resolution strategy, they have access to values in JavaScript scope, which means they can just use normal JavaScript imports. What's more, they can define other local components, helpers, or modifiers and export them or not as makes sense. They can do the same kind of extraction refactors they do with JavaScript or CSS. And other tools from the JavaScript ecosystem “just work”—from custom CSS tooling to GraphQL queries authored with [Apollo Client’s `gql` template strings][gql] and anything else the ecosystem comes up with.

[gql]: https://www.apollographql.com/docs/react/data/queries/

At the same time, since the body of a template defined with a `<template>` tag has all the same rules as Glimmer templates do today, this new authoring format keeps all the goodness of today’s clear separation of concerns between HTML and JavaScript and CSS. That means it continues to empower developers who are HTML and CSS experts and reach for JavaScript only secondarily. Indeed, the design goes out of its way to make HTML/Handlebars-only files feel like first-class citizens.

Finally, introducing `<template>` completely unifies the story between app and test code: in this new world, introducing a test-only component is as simple as introducing any other component in the same file as an existing component.

In sum, `<template>` resolves each problem outlined above, and introduces new capabilities to boot.


### Constraints

There are a number of solutions which could address these needs and add these capabilities. This RFC proposes `<template>` out of all the possible options because I take the following constraints as guiding the design decision (and the ordering here is purposeful—items earlier in the list I judge to be more important than items later in the list):

1. Our choice of design **must not regress our ability to write tests**, and if it is possible to *improve* our testing story, we should take the opportunity to do so.

2. In the absence of hard technical constraints forbidding it, we should **prefer the solution which has the best story for teaching**—at all levels, including beginners but also supporting advanced users. In particular, this means that we should value both *progressive disclosure of complexity* and the *principle of least surprise*, and that we may need to weight them against each other, but that we should pay particular attention when they agree.

3. This design must cleanly interoperate with existing Ember codebases. That is, adopting this **must not require users to migrate their entire codebase at once**.

4. We should **prefer a design which provides more flexibility** to end users over a design which provides less.

While it is certainly possible to differ with these constraints *a priori*—reevaluating constraints is, in a very real sense, [how we got to this very RFC](https://github.com/emberjs/rfcs/pull/367#issuecomment-423839940)—we also run the risk of paralysis if we *continually* reevaluate from first principles. More challenging is inevitable disagreement about how we *weight* these constraints. On that front, there is no possibility of *final* agreement, but we should commit to some ordering for the purposes of this design so that the rest of it can proceed on the same terms.


## Detailed design

Introduce a new high-level syntax, the `<template>` tag, which is syntactical sugar for `setComponentTemplate` and `precompileTemplate`, in conjunction with the existing Ember and Glimmer `Component` classes and the special template-only component class returned by the `templateOnlyComponent` default export from `@ember/component/template-only`. There are three distinct, legal forms for this compilation:

- a standalone `<template>` at the top level of a module
- a `<template>` bound to a name
- in the body of a component class

For a discussion of the `setComponentTemplate` and `templateOnlyComponent` primitives, see [RFC #0481][rfc-0481]; for discussion of the `precompileTemplate` primitive, see [RFC #0496][rfc-0496]. This discussion will *assume* rather than *define* those.

[rfc-0481]: https://emberjs.github.io/rfcs/0481-component-templates-co-location.html


### Compilation

The value produced by authoring a `<template>` is a *JavaScript value*, and accordingly may be exported, bound to other values, passed as an argument to a function or set as a value on a class, etc. However, the value may not be treated as *dynamic*: the rendering engine will evaluate the value exactly and only once. Therefore, it is nonsensical to use it with a `let` binding or to attempt to create new `<template>`s dynamically from the body of a function. (A function may of course return different components based on its arguments, etc.; it simply may not *construct* new component templates dynamically.)


#### Bound to a name

Given a `<template>` tag assigned to a name in JavaScript scope:

```js
const Greet = <template>
  <p>Hello, {{@name}}!</p>
</template>
```

Then the compiled output is:

```js
import { precompileTemplate } from '@ember/template-compilation';
import { setComponentTemplate } from '@ember/component';
import templateOnlyComponent from '@ember/component/template-only';

const Greet = setComponentTemplate(
  precompileTemplate(`
  <p>Hello, {{@name}}!</p>
  `
  ),
  templateOnlyComponent()
);
```

If the `<template>` references values in scope, they will be included in an object with a `scope` argument (matching the current implementation of the underlying primitives). Thus, this definition—

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

const Greet = <template>
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

const Greet = setComponentTemplate(
  precompileTemplate(`
  <p>Hello, {{@name}}!</p>
  {{#if (isBirthday @dateOfBirth)}}
    <p>Happy birthday! 🎈</p>
  {{/if}}
  `,
    {
      scope: () => ({ isBirthday }),
    }
  ),
  templateOnlyComponent()
);
```

**Two important notes:**

1. Users *should* always and only use `const` bindings for the result of such a template, because Ember will never reevaluate if the name is re-bound later. (Even if we wanted to do that, it would be difficult at best: nothing would notify Ember that it *should* re-evaluate that value!) We should introduce a lint rule forbidding assignment of a `<template>` to a `let` binding to prevent that confusion.

2. Relatedly, in normal app code, authors should not introduce component definitions `<template>`s in contexts where they will be “re-executed,” e.g. in a function body. It is technically possible to create components from a function, like so:

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

    The problem here is that this requires re-running both the template precompilation step *and* the creation of the template-only empty backing instance which has `null` for `this`. Re-doing the template precompilation will *work*, but it is expensive and slow.

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


#### Standalone

The compiled output for a top-level `<template>` tag *not* bound to a name is a default export, but otherwise identical to a `<template>` declaration bound to a tag. Thus, given this input:

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
  `
  ),
  templateOnlyComponent()
);
```

Scoped values are included the same as with named definitions.

Since the compiled output is a default export, it is a static error to have multiple top-level (i.e. not bound to a name) `<template>`s in a file—because it is a static error to have multiple `export default` statements in a JavaScript file. (We should provide linting tooling to error on this case, rather than letting it fail at runtime.)


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
  precompileTemplate`
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
  `
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

That is because the compilation output does *not* embed the template in the class' body in any way, but instead associates it *externally* to the class—but private class fields are only accessible within the body of the class itself, per the ECMAScript spec. While we could invest time to change the implementation to avoid this, it is not generally a problem. Because we do not interact with component class instances directly—only through the template layer—we do not actually *need* private class fields.

Even, this is a real gap, which we could address in a future RFC. Notably, it is *not specific to this proposal*, but applies to *all* proposals built on the current primitives.


### Interop

Since all existing components already work using the same low-level primitives the new system uses, strict-mode components using `<template>` can import and invoke components authored in Ember’s loose mode format. Similarly, since loose mode components are resolved to a component which is the default export of a module in the correct/conventional location on disk, components authored in strict mode with `<template>` and exported as the default export in that conventional location will be resolveable by loose mode components as well.

However, other components exported as *named* exports will not be available in loose mode. This is a temporary incoherence which will be resolved as the ecosystem migrates to strict mode. As a workaround, users may choose to create reexport-only modules to allow loose-mode access. For example, given a module `app/components/named-only.js` with two named export components:

```js
export const Greet = <template>
  <p>Hello, {{@name}}!</p>
</template>

export const Farewell = <template>
  <p>Hello, {{@name}}!</p>
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


### The “prelude”

While all values used in templates must be explicitly in scope, Ember[^glimmer-prelude] will provide some via a “prelude”.[^prelude] These are always in scope and do not need to be imported.

The helpers in the prelude are:

- `action`
- `array`
- `component`
- `concat`
- `debugger`
- `each`
- `each-in`
- `fn`
- `get`
- `has-block`
- `has-block-params`
- `hash`
- `if`
- `in-element`
- `let`
- `log`
- `mount`
- `mut`
- `on`
- `outlet`
- `query-params`
- `unbound`[^unbound-deprecated]
- `unless`
- `yield`

The modifiers in the prelude are:

- `on`

The components in the prelude are:

- `LinkTo`
- `Input`
- `Textarea`

[^glimmer-prelude]: Glimmer.js may provide its own prelude. While long-term the two should likely align, this RFC simply takes the _status quo_ as a given

[^prelude]: “Prelude” is the conventional name for this functionality in programming language design.

[^unbound-deprecated]: `unbound` is deprecated, but since it will not be removed till Ember 5.0.0, it should appear in the prelude nonetheless.


### Tooling

To support the new format, we need to update tooling across the ecosystem to understand the format.


#### Syntax highlighting

First, we need syntax highlighting support across the ecosystem. Support already exists [for VS Code][vsc-highlighting], which represents the single largest group of web developers; as well as for any tool which can use [a tree-sitter grammar][tree-sitter-highlighting] (e.g. Neovim).

We will also need to implement support in [Linguist](https://github.com/github/linguist) for GitHub syntax highlighting.

Beyond that, we should encourage the community to add support for other editors (IntelliJ, Atom, Emacs, etc.) as well as for tools like [Rouge](https://rubygems.org/gems/rouge) (which powers GitLab syntax highlighting) and other highlighters, but need not treat those as *blocking* adoption of first-class component templates.

[vsc-highlighting]: https://marketplace.visualstudio.com/items?itemName=chiragpat.vscode-glimmer
[tree-sitter-highlighting]: https://github.com/alexlafroscia/tree-sitter-glimmer


#### Blueprints

All blueprints will need to be updated to support generating the new format. During the transition period, we should allow generating both. The rollout will follow the example of the rollout of Glimmer components with Octane:

1. Introduce the ability to author components in the new format with a new `--strict` flag, but leave the default today’s loose mode format. Introduce `--loose` as an explicit flag for using today’s loose mode format.

2. When Ember Polaris is released, change the default to the new format, while leaving loose mode available via `--loose`, and preserving `--strict` as an explicit flag for the new default.

3. If or when loose mode templates are deprecated, the supporting blueprint infrastructure can be removed, including the `--loose` flag.

The current blueprints support generating a backing class for any existing component template which does not already have a backing class with the `component-class` format. We have two choices about the behavior of that blueprint for strict mode templates:

1. Do not support it. Adding a backing class is simply a matter of adding an import and adding a class.

2. Reimplement the blueprint using an AST transform (which we have prior art for: route generation uses that approach), to add a backing class for an existing default export in the module.

We should take (1) here as a default starting point, while encouraging the community to implement (2) if interested.


#### Linting and formatting

As with syntax highlighting, we need to support the new format with linting and formatting integration.

For linting, we need to make two changes:

1. Create an ESLint processor which uses the same Babel transform as the core behavior itself (as provided currently by [ember-template-imports][eti]) to make normal ESLint rules (e.g. unused imports and values) work with the template scope.

2. Going the other direction, make it possible to use the existing `ember-template-lint` rules in `.gjs`/`.gts` files. This likely means integrating `ember-template-lint` directly into ESLint, in much the same way that other sub-language integration is done (in the same way that e.g. [eslint-plugin-svelte3][eslint-svelte] integrates Svelte's custom language).

[eslint-svelte]: https://github.com/sveltejs/eslint-plugin-svelte3

For formatting, we need to implement a custom parser plugin and language which will make Prettier able to format both the host JavaScript files and the embedded templates. This will need to present a view of the non-template parts of the file to Prettier so that it formats the JavaScript correctly without updating the template contents, and vice versa. The primary work here is to make it so that we can leverage Prettier's existing support for JavaScript/TypeScript and Handlebars in a `.gjs`/`.gts` file (rather than simply ending up with a parse error, as happens when you try to treat those files as pure JS or TS).


#### Language server support

The final piece of tooling we need for supporting this is *language server support*. Language servers using the [Language Server Protocol][lsp] allow a variety of different editors (including e.g. Vim, Visual Studio Code, Emacs, Atom, Sublime Text, IntelliJ, and others) to use a single language server Currently, for uninteresting historical reasons, there are a handful of language servers floating around which Ember developers use. Most important for our purposes are the [Unstable Ember Language Server][uels] and [Glint][glint].

**Neither of these is technically a hard blocker for adopting first-class component templates, but we expect there to be significant community demand for support.** However, the existing support in these tools for the `hbs` experiment means that supporting `<template>` is relatively straightforward: the work needs to be done, but is not especially large. In particular, the same Babel transform which makes `<template>` work and can power ESLint and Prettier integration should provide the necessary information for language servers as well, which can then leverage their own interpretations of templates (e.g. Glint's mapping from a Handlebars template to a TypeScript representation) to provide richer feedback, autocompletion, go-to-definition, documentation hovers, etc.

[lsp]: https://microsoft.github.io/language-server-protocol/
[uels]: https://marketplace.visualstudio.com/items?itemName=lifeart.vscode-ember-unstable
[glint]: https://github.com/typed-ember/glint


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

3. When refactoring to use a getter for `src`, use the value as `${TOKEN}`. Note explicitly how there is a symmetry between JS and templates: you can use whichever approach is clearer for a given context.


##### Reusable Components

This section provides is an opportunity to show how useful it can be to introduce local functions. The code samples here currently use a backing class, but they only do so to provide a home for getters which provide an encoded URI for the Mapbox token and derive the `src` from the arguments.

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

At this point we could *additionally* show that we could introduce a backing class, and discuss the tradeoffs of introducing a class when we don't have any other local state. This also allows us to encourage just using functions unless you *do* need local state.

The section “Getting JavaScript Values into the Test Context” will also be possible to simplify: we will simply be able to introduce tracked state locally and update it directly, without special testing helpers. That will dramatically reduce the number of bespoke ideas we have to cover here. Much of the related work will be addressed in other RFCs, but being able to use the *same* primitives to bring values into scope for tests as we do in apps (immediately above!) will be very helpful in reducing what we have to cover in this section.


#### Core Concepts: Components

This entire section will also need to be substantially reworked. Once again, I am here summarizing the changes rather than trying to rewrite the guide in place. Each section represents a page to be changed; if a section is not mentioned, it needs no substantive changes—likely only switching over to using the `<template>` wrapper.

At some point in the course of this discussion, we should call out (e.g. with a “Zoey says” block) that users should treat `<template>` the same way they treat a costly function which produces a result for the life of the whole app, and should therefore avoidng using `<template>` in function bodies rather than hoisting them, etc. This cannot be a hard and fast rule about where `<template>` definitions live, because there are plenty of ways to do it safely, and what’s more we *need* to do it in test modules.

The point is simply to align people’s mental model for `<template>` with *other* costly operations, since these concerns are not specific to component creation.


##### Introducing Components

Unsurprisingly, this is the section which will see the most sweeping changes.

- As described in the tutorial, our introduction will depend on the design chosen for route templates. We will either note that we’ve already seen our first component, if the application template *was* a component, or note the similarities and differences between route templates and components otherwise.

- We can continue to show breaking the component out into separate files, with a top-level `<template>` (serving, so far implicitly, as a default export).

- Then, back in the application file, we can show using `import` to refer to it.

- As in the tutorial, the discussion around naming will need to be updated to indicate that we capitalize by *convention*: it will no longer be a hard requirement. Likewise, the “Zoey says...” will go away because we will no longer be using resolution to get imports.

- After showing the other extraction-style refactors, we can show how components which don't need to be exported can just be defined locally with a `const` declaration, and explain that the standalone `<template>` tag is sugar for a default export. This will also provide the first hook for defininig helpers etc. locally in following sections.

- We will entirely drop the folder namespace syntax (`::`) discussion, in favor of showing how normal JS imports handle that concern—including showing how the combination of named exports and namespace-style imports handle those. (This will necessitate reworking the example, which currently uses that namespacing as a means of scoping.)


##### Helper functions

Instead of introducing `app/helpers` and the resolution-based lookup, we can introduce the helper as a local function in the component which needs it. This will be the first place where this guide explicitly calls out that components have access to values in their surrounding scope, just like normal JavaScript. This will be a good point to call out the power and versatility this affords.

Additionally, instead of the *next* section being the place where we first identify that JS is needed to make our UI dynamic, we will address that here. The next section can then build on that by showing how classes make certain patterns *easier*.


##### Component State and Actions

Here, the content will need to shift in two ways:

1. The motivation for introducing a backing class shifts slightly: we have the ability to have state at the module level already, including via class-backed helpers. What we need is a way to have state that is for *just one component*. A class is JavaScript’s first-class way of doing that, so we have a version which supports it.

2. Having made the motivation clear, we can show the `<template>` in the body of the class and explain that it is exactly the same as a standalone template component, except that it now has access to the backing class for local state, "actions", etc.


##### Template Lifecycle, DOM, and Modifiers

Once again, many of the changes here will be mechanical: just using the new syntax. However, this also provides another opportunity to discuss (and demonstrate) the value of local-only vs. exported functionality. Both of the main custom modifier examples here currently show highly-reusable examples of modifiers which *should* be exported and should indeed probably live in their own modules. Accordingly, we might find an example which shows the value of having a locally-scoped modifier—e.g. something which manages the private details of an `iframe`.


### API Docs

There is presently no API for `<template>` itself, as proposed in this RFC, though it leaves room for future RFCs to do so.[^emblem-etc] Since there is no import location, we should cover it under the `@glimmer/component` module documentation. This will be a natural home for it, since we will always discuss it in the context of components.

[^emblem-etc]: Historically, for example, many Ember apps used [Emblem][emblem] as a templating language—and it is still possible to do so today! In the future, that could be supported with `<template lang="emblem">`. This would also be an easy home for experiments with a Svelte-like syntax with e.g. `<template lang="svelte">` etc.

[emblem]: http://emblemjs.com


### TypeScript

The type of a component is not affected by this proposal. However, it is worth seeing *how* a component defined using `<template>` works with types, at least for the purpose of documentation (and for integration with the current DefinitelyTyped definitions).

- For a class-backed component, there is no change to the *types* of the component when using `<template>`. As described above in the discussion of language servers, tools like Glint will need to provide an interpretation of the body of a `<template>` which correctly understands the scope in which it is embedded, i.e. correctly providing `this` to it.

- For a template-only component, defining the type will require a type import to represent that there is a component with no `this` context, etc. Glint already supplies such a type, albeit with the types updated for [RFC #0748][rfc-0748]. For today’s purposes, we can simply augment the [existing types on DefinitelyTyped][DT-toc] with `Args`.

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

    Users can then define a template-only component like this:

    ```ts
    import { TC } from '@glimmer/component';

    const Greet: TC<{ name: string }> = <template>
      <p>Hello, {{@name}}!</p>
    </template>
    ```

    While this empty interface is currently more or less useless from a type-checking perspective (we will need something like Glint to support it), it suffices to provide a hook for documentation tooling such as TypeDoc or API Extractor—but this suffices for the level of support we have for TypeScript today.

[rfc-0748]: https://github.com/emberjs/rfcs/pull/748
[DT-toc]: https://github.com/DefinitelyTyped/DefinitelyTyped/blob/54d540ab4deb2588c0eff39dadf370cbf0a2dee4/types/ember__component/template-only.d.ts


### Existing Ember users

The Ember community has long experience with the idea that we can only have *one* component per file, and that component templates and the backing class must always be in separate files. The name of this feature, *first-class component templates*, is designed to help explain how it relates to that historical experience. In the past, templates in many ways were second-class citizens of the overall experience of authoring an Ember app—especially for template-only components. Adding even a small amount of functionality to a template came with a lot of friction and other downsides: switching to a class-backed component, or introducing a globally-available helper. Now, adding a local helper utility is no different for a template-only component than it is anywhere else: write a function!

As a result, we can teach this to existing Ember users as taking everything they already know about how templates work, while making it faster, easier, and lighter-weight to solve common problems. The one significant new concept here is the `<template>` tag. We can describe that to existing features as the one addition which raises templates up to being a first-class tool in your app or addon.

A blog post can introduce the feature along these lines when the feature ships, with examples showing how it simplifies existing patterns and enables new capabilities. For simplifying existing patterns, we might pull the same Mapbox example from the guides shown above. For enabling new capabilities, we could show how it enables using native CSS Modules with none of the hacks required by Ember CSS Modules.


## Drawbacks

- Since there is no notional import for `<template>`, there also isn’t a notional home for API documentation for it other than components.

- We must build a custom tooling integration with Prettier for the file format to *parse*. (As discussed below, we must build custom tooling to *use* Prettier for other options, but Prettier can *parse* them without custom tooling.)

- While not especially important in the context of a front-end framework (as opposed to a framework-less/“vanilla” context), `<template>` does technically overlap with a platform built-in, and would look very strange if a user *did*  want to use the built-in form.

- Developers may put an unreasonable number of components, helpers, modifiers, etc. in a single file, degrading the maintainability of that module. However, the counterpoint here is that large files are already common in many code bases, with or without this tool. Indeed, that happens in non-UI and UI code bases alike!

    Moreover, experience from frameworks which restrict component authoring formats to a single component per file, including Ember’s loose mode templates as well as Vue and Svelte SFCs, is that those components themselves tend to balloon in size. Sometimes that’s because everything in those components is notionally related or because much of it should be treated as "private API" for that component (even if it would be helpful to refactor small local components). Sometimes it is just because of the annoying overhead of needing to create a separate file to break the huge component into smaller pieces, and then import them all (or make them globally available, in Ember loose mode template!).

    The analogy here would be if a JavaScript module could only have a single function or class in it, or a CSS file could only have a single declaration in it, regardless of what actually made sense for that particular module.

- The syntax offered here, `<template>`, overlaps with [a platform built-in][platform-template]. This may provoke some degree of confusion for users if they are familiar with it. However, there are several reasons to think this drawback is not significant:

    - In practice `<template>` is very-little used, and only in the context of progressive enhancement with vanilla JS—not with frameworks.

    - Although it looks a little odd, the platform-native `<template>` can still be nested *within* a `<template>` tag as defined here.

    - Other frameworks (most notably Vue) have used `<template>` in much the same way we are here with no major confusion on the part of developers.

    - Most importantly, there is no *actual* conflict with the platform built-in, since `<template>` is not *JavaScript* syntax, which is where we are using it.

    - As a bonus: in a certain sense, the use of `<template>` here “rhymes” with the version from the platform: it represents the dynamic HTML content associated with some JavaScript functionality.

[platform-template]: http://developer.mozilla.org/en-US/docs/Web/HTML/Element/template


## Alternatives

Within the major strokes of this design proposal, we could tweak the invocation for the template space to clarify that it does *not* overlap with the built-in `<template>` tag.

- Use `<Template>` or `<Glimmer>` or similar. This would disambiguate it from the built-in `<template>`, but would introduce ambiguity with component invocation.
- Use a new sigil—much as we use `<:main>...</:main>` for named blocks, we could do `<[template]>...</[template]>` or something similar. While verbose and not especially pretty, this avoids overloading the platform tag.

Additionally, there are three major alternatives which Ember community members have proposed in the design space:

- **imports-only:** a design which uses “front-matter” to add imports, and only imports, to templates, while maintaining
- **single-file components (SFCs)**: a design which follows the example of Svelte and Vue and make HTML the basis of a component, and
- **`hbs` template literals**: a design which mirrors the `<template>` design quite closely, but uses `hbs` template literals similar to those we use in tests today

I discuss each of these briefly below; for a *much* longer and more thorough discussion, please see the \~16,000-word series of blog posts I wrote as a deep dive: [**Ember Template Imports**](https://v5.chriskrycho.com/journal/ember-template-imports/).


### Imports-only

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

    export default modifier((el, _ , { with: newUrl }) => {
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

The major upside to this is that it is the smallest possible delta over today’s implementation. However, it has a number of significant downsides

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

Perhaps most critically, this is **much worse than the _status quo_ for tests**. It requires that we do one of the following:

- Require people’s test authoring format to become *massively* more verbose and less useful, with imports in every single test `hbs` string, to support strict mode for tests:

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

          <ComponentToTest @anArg= />
        `);
      });
    });
    ```

    The first thing to notice is that there is no way to import `ComponentToTest` here just once. This overhead will multiply across the number of items to reference in a given test—every component, every helper, every modifier!—as well as across the number of tests. This is a *large* increase in the burden of testing compared to today.

    The second thing to notice is that this also requires us to maintain, *and to teach*, the `hbs` handling for tests (or to design some replacement for it), on top of the “regular” template handling for components. This is the same situation as in Ember apps today—but since first-class component templates allow us to *improve* the consistency between app code and test code, this counts as a negative by comparison!

- Continue to support a completely separate design for testing than for app code. In that case, though, supporting strict mode templates in tests *and* avoiding the verbosity of the first option means we need a separate authoring format for tests—in fact, it basically requires that we fully implement something like the first-class component templates design!

[^recursive-module]: To see this for yourself, follow the instructions in [this gist](https://gist.github.com/chriskrycho/dc5adabd80c04d405c7a4894c0ffb99e). I had to test this out myself, and while it’s actually *very good* that modules work this way, I was initially surprised by it! If you’re curious: imports and exports are *static* and so are analyzed before the module is executed.

[^recursive-import-perf]: Without additional post-processing, this would also introduce extra runtime overhead in terms of the imports!


### SFCs

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

First, SFCs do not allow multiple components to be defined in a single file. This has been an ongoing sticking point with the Vue and Svelte designs, such that there is even an [open-for-years RFC for Svelte for supporting at least a subset of this functionality](https://github.com/sveltejs/rfcs/blob/inline-components/text/0000-inline-components.md).

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

      <ComponentToTest @anArg= />
    `);
  });
});
```

Besides having all the same problems as the imports-only approach does, notice that this also substantially increases the cost of tooling even at the level of syntax highlighting, because now we need multiple nested layers of syntax highlighting: `hbs` strings include both HTML and JavaScript! While some syntax highlighters support this, it is a much higher lift for those which do not.

And, once again, avoiding those problems more or less requires that we fully implement first-class component templates to avoid this!


### Template literals (`hbs`)

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

#### Advantages

This has a couple significant advantages!

First, unlike with the `<template>` proposal, Prettier can *parse* a file using `hbs` string with no further changes. (It cannot *format* them, however: it treats the contents of the string opaquely.) This is a small, but significant, decrease in the cost of supporting the format both up front and over time.

Second, it feels familiar to developers in the Ember ecosystem used to using `hbs` strings with their tests.

Third, the broader JavaScript ecosystem makes use of a number of template string syntaxes, e.g. with `css` from [Emotion][emotion] or [`gql` from Apollo][gql], so it has familiarity for people coming from *outside* the Ember ecosystem as well.

Fourth, it shares many of the other strongly-positive properties of the `<template>` design, including that it works exactly the same way for testing and app code.

These advantages are strong enough that this is *absolutely* the second-best move in the design space for us, and given a choice between maintaining the status quo and using `hbs` (i.e. if `<template>` were off the table), ***we should absolutely choose `hbs`***.


#### Disadvantages

Those positives notwithstanding, it also has some significant disadvantages compared to `<template>`.

First, the design repurposes JavaScript syntax and gives it totally different semantics—like `<template>` does with HTML, but with *zero* signal from the context that it is doing something special.

- `hbs` is *not* actually a template string literal; it is a compile-time macro. Attempting to use it like a template string literal (e.g. by using `${...}` string interpolation) is a build-time error. This substantially undercuts the familiarity of the design: `css` and `graphql` and similar are *actual* string templates, not compile-time macros, and accordingly developers can use normal JavaScript semantics with them.

- The use of `static template = ...` has the wrong semantics: static class fields do not have access to an instance’s `this`—but templates quite expressly *do*. The whole point of a component with a backing class is to provide a normal JavaScript `this`, so this is a significant mismatch, which has consequences for both teaching and tooling.

Second, the learning path is *much* less gradual: the simplest possible component requires showing and at least minimally explaining JavaScript import and export semantics and template literals *and* that it isn’t a normal template literal as described above.

Third, explaining what exactly `hbs` invocations produce is also strange: they aren’t actually JavaScript expressions, but they *appear* to be. In a template-only context, “invoking” `hbs` produces a component; in a class, it produces the *template* for that component. This is the same as with the `<template>` proposal, but it has the additional quirk of using JavaScript syntax to do it, rather than shifting languages.

Net, while there are some nice features to the `hbs` proposal, it comes out significantly worse than `<template>` in most ways we care about. The decreased tooling costs are real, but they are much smaller than the other downsides of the format.


## Unresolved questions

- Does the possible confusion with the platform `<template>` warrant adopting an alternative syntax, whether component-like (`<Template>`) or using an additional sigil (`<[template]>`, `<% ... %>`, `<$ ... $>` etc.)? If so, what design? Here we must keep in mind that the design should not be ambiguous with “dynamic” behavior (e.g. `<{template}>` which is suggestive of the Svelte and React expression marker, and which we might find attractive for future iterations of template language ourselves).

- Are `.gjs` and `.gts` the best file extensions?

- How does this relate to the currently un-merged [RFC #0731: Add `setRouteComponent` API][rfc-0731]? That is: can we merge this and proceed with authoring components while there is an unresolved design problem for the related issue of routes, controllers, and their host components? Or should we see this as a helpful part of resolving *that* design question? (I believe we can move forward in parallel.)

    The primary challenge here is that, as things stand, our guides would have some fairly substantial incoherence until we solve the problems which #0731 is addressing: route templates would be totally different from component templates in their semantics and behavior. This points to the ongoing and increasing divergence of the `Route` and `Component` design from the rest of the framework, but it’s directly connected to *this* RFC pedagogically.

- Should we include a plan for a staged rollout of deprecating namespace resolution in this RFC, rather than tackling it in a later RFC?

[rfc-0731]: https://github.com/emberjs/rfcs/pull/731

<!--

> Optional, but suggested for first drafts. What parts of the design are still
TBD?

 -->