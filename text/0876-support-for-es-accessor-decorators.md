---
stage: accepted
start-date: 2022-11-27T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - framework
  - learning
  - typescript
prs:
  accepted: https://github.com/emberjs/rfcs/pull/876
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

# Support for ES (`accessor`) Decorators

## Summary

Introduce support for the [ECMAScript Decorators Proposal][spec] for the key decorators which are part of Ember’s modern programming model, `@service` and `@tracked`:

```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking/accessor';
import { service } from '@ember/service/accessor';

export default class Example extends Component {
  @service accessor someService;
  @service('named') accessor anotherService;
  @tracked accessor someLocalState = 123;
}
```

We will provide new imports to support these, a codemod to migrate to the new form, and guidance for both app and addon users about how to manage the transition.


## Outline <!-- omit in toc -->

- [Summary](#summary)
- [Motivation](#motivation)
- [Detailed design](#detailed-design)
  - [The change to the spec](#the-change-to-the-spec)
  - [Path forward](#path-forward)
  - [Updated decorators](#updated-decorators)
    - [`@tracked`](#tracked)
    - [`@service`](#service)
  - [Excluded legacy decorators](#excluded-legacy-decorators)
  - [Migration path](#migration-path)
    - [Codemod](#codemod)
    - [Lint rule](#lint-rule)
    - [Apps](#apps)
    - [V2 (Embroider) Addons](#v2-embroider-addons)
    - [V1 (Classic) Addons](#v1-classic-addons)
- [How we teach this](#how-we-teach-this)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Unresolved questions](#unresolved-questions)



## Motivation

The [ECMAScript Decorators Proposal][spec] has advanced to Stage 3 in the JavaScript stabilization process, and will have full support in both Babel and TypeScript when TypeScript 5.0 ships in early 2023. It is important that Ember be aligned with the ECMAScript spec, and our users will expect to be able to use these new features. Although the Stage 3 form is available in Babel today, we expect that official and widely publicized support for the updated decorators spec in TypeScript is going to be a watershed moment. Ember users rightly expect to be able to use new language features as they stabilize, so it is important that we support these as soon after TypeScript supports them as possible.

---

🤔 *Why is TypeScript, and not just Babel support, a key factor?* We have committed to officially support both JavaScript and TypeScript, per [RFC 0724][0724]. Accordingly, we cannot really support a feature without a path to supporting it for both JavaScript *and* TypeScript, and the TypeScript team's plan for decorators was not clear or stable until lately as part of the lead-up to TypeScript 5.0. Net, we are gated on TS in *this* case because its support arrived later; in the future it could be vice versa. The key is not TypeScript support specifically, but the ability to support all our users.

---

While we expect the transition from the legacy (Stage 1) decorators proposal to take some time, the legacy transforms from Babel and TypeScript[^other-transpilers] will eventually be deprecated and removed, so we need to have a path away from them.

[spec]: https://github.com/tc39/proposal-decorators
[0408]: https://rfcs.emberjs.com/id/0408-decorators
[0440]: https://rfcs.emberjs.com/id/0440-decorator-support
[0724]: https://rfcs.emberjs.com/id/0724-road-to-typescript/

[^other-transpilers]: and other transpilers like SWC and ESBuild, which will be increasingly viable for Ember users as we migrate to Embroider


## Detailed design

When we originally [introduced support][0408] for decorators, we [specified][0440] our support policy and approach for the Stage 1 decorators:

> Stage 1 decorators will be considered a first class Ember API. They will be
supported until:
> 
> 1. A new RFC has been made to introduce a new decorator API as a parallel API to the stage 1 decorators.
> 2. A deprecation RFC has been created for Stage 1 decorators.
> 
> They will follow the same deprecation flow, and same SemVer requirements, as any other Ember feature.

This RFC is the first step in that process: it introduces a new decorator API as a parallel API to the stage 1 decorators. A follow-on RFC will separately deprecate the existing Stage 1 decorator APIs.


### Background

The current basic form of property decorators looks like this:

```js
import Component from '@glimmer/component';
import { tracked } from '@glimmer/tracking';
import { service } from '@ember/service';

export default class Example extends Component {
  @tracked localState;
  @service session;
}
```

The spec changed both the syntax and the semantics of these kinds of decorators. The semantics are easy enough for us to change but **do** require a new implementation (discussed below). The syntax is also a straightforward change, simply requiring the addition of the new `accessor` keyword:

```js
export default class Example extends Component {
  @tracked accessor localState;
  @service accessor session;
}
```

This `accessor` keyword is exactly equivalent to having a `get` and `set` pair with a backing private field (but with a name that is invisible), like this:

```js
// this:
class Example {
  accessor foo;
}

// compiles to roughly this:
class Example {
  #foo_1234asdfLOL;

  get foo() {
    return this.#foo_1234asdfLOL;
  }

  set foo(value) {
    this.#foo_1234asdfLOL = value;
  }
}
```

Decorators can then layer on top of this. This is more or less what `@tracked`, `@service`, etc. already do—but a spec decorator gets a very different argument, and behaves fairly differently than the legacy decorators implementation.

We *could* try to absorb that with a single overload, but that has a much higher risk of bugs *and* doesn’t give us any hooks for linting etc. Linting matters because we want to be able to tell people statically (not just with annoying Babel compile errors) that they *must* use `accessor` with this version instead.

We also want to make sure that we leave room for alternatives in the Polaris timeframe. For example, perhaps we ship something like this:

```js
import { reactive, service } from '@glimmer/reactivity';

class Example {
  @reactive accessor localState;
  @service accessor session;
}
```

We want to make sure not to couple *this* change—which just unlocks using the spec decorators—to *that* possibility, which isn’t finalized. We also want this to be trivially codemoddable, and we also plan to deprecate the existing decorators in a future deprecation RFC, so what we ship here is certainly only a transitional path: eventually, regardless of any `@glimmer/reactivity` package, the *primary* imports for these packages should be ES decorators, *not* the legacy decorators.


### Updated decorators

Accordingly, we will add a new location to import these decorators: a `<path>/accessor` variant of the current import location.

```js
import { tracked, cached } from '@glimmer/tracking/accessor';
import { service } from '@ember/service/accessor';
```

These decorators will have *exactly* the same semantics for Ember users as the Stage 1/legacy decorators do; the only difference is their import location and the syntax with which they must be applied. Note that they 

In JavaScript:

```ts
import Component from '@glimmer/component'
import { tracked, cached } from '@glimmer/tracking/accessor';
import { service } from '@ember/service/accessor';

export default class Example extends Component {
  @service accessor basicExample;
  @service('explicitly-named') accessor explicitlyNamed;
  @service('namespaced@explicitly-named') accessor nameSpacedToo;

  @cached
  get expensiveDerivedState() {
    // ...
  }

  @tracked accessor count = 0;
}
```

**Note:** The `@cached` decorator has the same *syntax* when applied to a getter as it does with legacy decorators, but different *semantics*.

There is one additional key difference in TypeScript compared to today's decorators. TypeScript users with the Stage 1 decorators [*must* use the `declare` modifier][declare-blog]; with the Stage 3 decorators they *cannot* use `declare`. Here's how that works:

```ts
import Component from '@glimmer/component'
import { tracked, cached } from '@glimmer/tracking/accessor';
import { service } from '@ember/service/accessor';
import type BasicExample from '../services/basic-example';
import type Named from '../services/named';
import type NamespacedToo from 'namespaced/services/namespaced-too';

export default class Example extends Component {
  @service accessor basicExample: BasicExample;
  @service('named') accessor explicitlyNamed: Named;
  @service('namespaced@named') accessor nameSpacedToo: NamespacedToo;

  @cached
  get expensiveDerivedState() {
    // ...
  }

  @tracked accessor count = 0;
}
```

Effectively, `accessor` does the work that `declare` was doing previously in telling TypeScript that the item could be depended on to be present, but it does it in a non-TypeScript specific way.

[declare-blog]: https://jamescdavis.com/declare-your-injections-or-risk-losing-them/


### Other decorators

We are *not* supplying imports for any other decorators. The list of decorators we will *not* be updating to this form:

- `import { inject as controller } from '@ember/controller';`: controller injection is an anti-pattern, and we do not want to add new code to support it.

- `import { computed } from '@ember/object';`: computed properties should be replaced with `@tracked` and native getters. This also means we will not include any of the computed property macro decorators from `@ember/object/computed`.

- The use of `tracked` in classic classes. The existing `tracked` decorator can be used in Classic classes like this:

    ```js
    import EmberObject from '@ember/object';
    import { tracked } from '@glimmer/tracking';

    export default EmberObject.create({
      someProp: tracked(),
    })
    ```

    While this was a useful bridge to allow opting into auto-tracking independent of migrating to classic classes, we do not want to add this additional complexity to *brand new* code.

- The decorators supplied by [ember-decorators](https://github.com/ember-decorators/ember-decorators), e.g. `@attribute`, `@className`, etc.

In all these cases, users should update to using Octane idioms instead *first*, then migrate to using the `accessor` decorators.


### Migration path

💡 **Big picture:** Legacy decorators support continues to exist in both Babel and TypeScript. However, decorator compilation is a *package-level* concern, which means it cannot be done at any level more granular than that.

The overall migration path will work like this:

- Introduce the new decorators into Ember itself.

- Supply a codemod, which does four things:
    1. Validate that you have sufficiently recent versions of Babel, the Babel decorators plugin, and TypeScript (as appropriate for your build setup).
    2. Updates all imports to use the new decorators.
    3. Updates all usage sites to add `accessor`.
    4. Updates your compiler settings (Babel *and* TS, if TS is present).

- Introduce a lint rule to make it easier for developers to remember not to mix the syntaxes.

    While using the new decorators in the wrong way will fail to work, giving a loud signal to users that something is wrong, we should also make it easy by shipping a new, on-by-default lint rule which will error under these conditions:

    - using one of the Stage 1 decorators with the `accessor` keyword
    - using one of the new Stage 3 decorators *without* the `accessor` keyword

    These two will make it so that any individual usage site is correct. However, given that compilation is a whole-package concern for any given app or library, we also need an additional lint rule forbidding the imports corresponding to modes which are not supported in the compilation mode they are currently using. For example: once users opt into using the Stage 3 decorators, the lint configuration should prevent them from having *any* uses of the Stage 1 decorators imported or in use.

    This will help people guarantee that their code bases are set up correctly end to end, without tripping over Babel compilation errors or runtime errors.

- After at least one LTS release (for the sake of addon maintainers, and per our usual approach), deprecate the old decorators. At a future major, remove them entirely. (Which major version that will be depends on when this all lands. Legacy mode decorators will definitely be supported till at least 6.0/November 2024!)

- Once we have the codemod ready, we can update the blueprints to default users into using the new decorators spec, and add the codemod to `ember-cli-update`'s codemods targeting that version or higher.

**Note:** The community will need to migrate any custom decorators as well to support this. Ember itself cannot enforce this, unfortunately!


#### Apps

Apps can opt into this at will, and do not require all of their addons to be in the new mode. The steps are:

1. Update to require a version of Ember which includes the new decorators. (This will be a breaking change, the same as any such bump.)

2. Update to a version of `ember-cli-babel` which supports the new decorators in the library's `dependencies`. `ember-cli-babel` will explicitly require a recent enough version of the Babel plugin to make this reliable.

3. Explicitly specify the plugin in your addon config's `'babel'` key. For example, in a fully-classic setup (not using `babel.config.js` at all), you might do this in your `ember-cli-build.js`:

    ```js
    // ember-cli-build.js
    let app = new EmberApp(defaults, {
      babel: {
        plugins: [
          {
            ...require.resolve('@babel/plugin-proposal-decorators'),
            version: "2022-03"
          }
        ]
      }
    });
    ```


#### V2 (Embroider) Addons

Since v2 addons have a publish step, they have a straightforward way to make this transition:

1. Update to require a version of Ember which includes the new decorators. (This will be a breaking change, the same as any such bump.)

2. Add the latest version of `@babel/plugin-proposal-decorators` to the library's `devDependencies`.

3. Explicitly specify the plugin, including the version which matches the Stage 3 version of the decorator spec, in your `babel.config.js`, like so:

    ```json
    {
      "plugins": [
        ["@babel/plugin-proposal-decorators", {
            "version": "2022-03"
        }]
      ]
    }
    ```


#### V1 (Classic) Addons

Classic addons bring along their own build to be run as part of the host app which consumes, but the basic flow will look the same as for v2 addons, just with a change in where/how the config is specified.

1. Update to require a version of Ember which includes the new decorators. (This will be a breaking change, the same as any such bump.)

2. Update to a version of `ember-cli-babel` which supports the new decorators in the library's `dependencies`. `ember-cli-babel` will explicitly require a recent enough version of the Babel plugin to make this reliable.

3. Explicitly specify the plugin in your addon config's `'babel'` key. For example, in a fully-classic setup (not using `babel.config.js` at all), you might do this:

    ```js
    module.exports = {
      options: {
        babel: {
          plugins: [
            {
              ...require.resolve('@babel/plugin-proposal-decorators'),
              version: "2022-03"
            }
          ]
        }
      }
    };
    ```


## How we teach this

There are three major teaching changes we need to make in support of this:

1. We need a clear blog post covering this when it is released, explaining both the reason for the change and the migration path.

2. We will need API docs for the new imports, and we should also add explicit text to the old imports' API docs indicating that users should migrate away from them and to the new imports.

3. Once the new forms are stable, we will need to update the guides to replace all uses of the legacy decorators with the new accessor imports and forms. As with the change itself, this will be a straightforward syntactical rewrite and update to the import locations.


## Drawbacks

There are no *new* drawbacks to supporting the ES decorators, as they are now on track to be fully supported part of JavaScript, the same as any other. The main drawback here is some degree of "churn" if we do introduce different versions of these decorators in Polaris, such as a hypothetical `import { service } from '@glimmer/resources';`.


## Alternatives

- Make the existing imports "just work" by adding yet more overloads. This has a serious problem: in that design, the function has to support both modes, but the actual compilation flow with Babel or TypeScript does *not* support both modes: you have to pick one, and your linting configuration has to become aware of that to help you do the right thing. It therefore has both a significantly higher risk of bugs in the implementation *and* a higher likelihood of end user frustration.

- Don't migrate. Stick with the Stage 1 decorators forever. This has huge, obvious downsides: we'd be diverging from standard JS, and eventually support will go away.

- Don't support a mixed mode world. Force a hard breaking change at either 5.0 or 6.0. This obviously makes the work to get through that major version *much* harder, and it's simply not how we do things for that very reason.

- Use a different import location, e.g.:

    ```js
    import { accessor as service } from '@ember/service';
    import {
      accessorTracked as tracked,
      accessorCached as cached
    } from '@glimmer/tracking';
    ```

    This is a shorter module import path but a longer import overall, and goes against the reasoning for switching to simply use `service` per [RFC 0752][0752], and will not work well with automatic import tooling powered by TS, JetBrains IDEs, etc.

[0752]: https://rfcs.emberjs.com/id/0752-inject-service


## Unresolved questions

- Does this RFC need to spec the corresponding changes for Ember Data's decorators as well? (This is a straightforward procedural question, not a technical question: adding `@attr`, `@belongsTo`, etc. can be done straightforwardly enough.)

- Are there other decorators we feel we *must* support for the transitional period? Obvious candidates include:

    - `import { inject as controller } from '@ember/controller/accessor'`

        We have not (yet) formally deprecated controller injection, so some apps may still be using it, and they will not be able

    - `import { computed, alias, ... } from '@ember/object/computed/accessor'`

        As with controller injection, we still support classic computed properties. However, Octane has been available for over three years, and we would very much prefer *not* to introduce additional support for what are by any reasonable definition legacy features. Moreover, if we support `@computed('key') accessor`, we should also presumably support *all* of the computed property macros as well (`@alias`, `@readOnly`, etc.). This would be adding a non-trivial amount of code to support patterns we want people to move *away* from.
        
        The strongest argument in favor here is that it allows us to decouple these migrations; but there is *already* a path to decouple them: people can finish the Octane migration and *then* switch to the new decorators form.
