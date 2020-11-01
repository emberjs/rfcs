---
Start Date: 2017-07-14
Relevant Team(s): Ember.js
RFC PR: https://github.com/emberjs/rfcs/pull/236
Tracking: https://github.com/emberjs/rfc-tracking/issues/26

---

# Summary

This RFC proposes to deprecate the prototype extensions done by `Ember.String`, deprecate the `loc` method, and moving `htmlSafe` and `isHTMLSafe` to `@ember/template`.

# Motivation

Much of the public API of Ember was designed and published some time ago, when the client-side landscape looked much different. It was a time without many utilities and methods that have been introduced to JavaScript since, without the current rich npm ecosystem, and without ES6 modules. On the Ember side, Ember CLI and the subsequent addons were still to be introduced. Global mode was the way to go, and extending native prototypes like Ember does for `String`, `Array` and `Function` was a common practice.

With the introduction of [RFC #176](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md), an opportunity to reduce and reorganize the API that is shipped by default with an Ember application appears. A lot of nice-to-have functionality that was added at that time can now be moved to optional packages and addons, where they can be maintained and evolved without being tied to the core of the framework.

In the specific case of `Ember.String`, our goal is that users that need these utility functions will include `@ember/string` in their dependencies, or rely on common utility packages like [`lodash.camelcase`](https://lodash.com/docs/#camelCase).

To achieve the above goal we will move the `isHTMLSafe`/`htmlSafe` pair into a new package, deprecate `String.prototype` extensions, and deprecate the utility functions under the `Ember.String` namespace.

The ["Organize by Mental Model"](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md#organize-by-mental-model) section of RFC #176 mentions the concept of chunking. In the current setup, `isHTMLSafe`/`htmlSafe` make sense in the `Ember.String` namespace because they operate on strings, and they are available on the prototype, `"myString".htmlSafe()`.
However, once prototype extensions are removed it becomes clearer that while this pair operates on strings, they don't transform them in the same way as `capitalize` or `dasherize`. They are instead a way for the user to communicate to the templating engine that this string should be safe to render. For this reason, moving to `@ember/template` seems appropriate.

Extending native prototypes, like we do for `String` with `"myString".dasherize()` and the rest of the API, has been falling out of favour more as time goes by.
While the tradeoff might have been positive at the beginning, as it allowed users access to a richer API, prototype extensions blur the line between what is the framework and what is the language in a way that is not benefitial in the current module-driven and package-rich ecosystem.

Relatedly, deprecating `Ember.String` and requiring `@ember/string` as a dependency allows Ember to provide a leaner default core to all users, as well as iterate faster on the `@ember/string` package if desired.
Doing this will also open a path to extract more packages in the future.

# Transition Path

It is important to understand that the transition path will be done in the context of the new modules API defined in RFC #176, which is scheduled to land in 2.16.
As this will likely be first of many packages to be extracted from the Ember source, the transition path arrived on needs to be clear and user-friendly.

## What is happening for framework developers?

The order of operations will be as follows:

1. Move `htmlSafe` and `isHTMLSafe` to `@ember/template`
   * Update https://github.com/ember-cli/ember-rfc176-data
2. Create an `@ember/string` package with the remaining public API
3. Create an `ember-string-prototype-extensions` package that introduces `String` prototype extensions to aid in transitioning
4. Make `ember-cli-babel` aware of the `@ember/string` package so it tells `babel-plugin-ember-modules-api-polyfill` not to convert those imports to the global `Ember` namespace
5. Update usages in Ember and Ember Data codebases so that the projects do not trigger deprecations
6. Deprecate `Ember.String`
   * Write deprecation guide which mentions minimum version of `ember-cli-babel`, and how/when to use `@ember/string` and `ember-string-prototype-extensions` packages
7. Deprecate `loc` in `@ember/string`

## What is happening for framework users?

If you are using `Ember.String.loc`, you will be instructed to move to a dedicated localization solution, as this method will be completely deprecated.

If you are using `Ember.String.htmlSafe` or `Ember.String.isHTMLSafe`, you will be instructed to run the [`ember-modules-codemod`](https://github.com/ember-cli/ember-modules-codemod) and it will update to the correct imports from the `@ember/template` package.

If you are using one of the other `Ember.String` methods, like `Ember.String.dasherize`, you will receive a deprecation warning to inform you that you should run the [`ember-modules-codemod`](https://github.com/ember-cli/ember-modules-codemod), update `ember-cli-babel` to a specific minor version, and add `@ember/string` to your application's or addon's dependencies.

If you are using the `String` prototype extensions, like `'myString'.dasherize()`, on top of the previous instructions you will be instructed to install `ember-string-prototype-extensions` in case updating the code to `dasherize('myString')` is not trivial.

## Timeline

* Deprecations are introduced - Ember 2.x
  * `String` protoype extensions are deprecated
  * `Ember.String` functions are deprecated
  * `loc` is completely deprecated
  * `isHTMLSafe` and `htmlSafe` are moved to `@ember/template`
* Transition packages are introduced - Ember 2.x
  * `@ember/string`, which replaced `Ember.String`
  * `ember-string-prototype-extensions`, which brings `String` prototype extensions back
* Deprecations are removed - Ember 3.x, `@ember/string` 2.x
  * New major version of Ember is released
  * New major version of `@ember/string` is released

# How We Teach This

## Official code bases and documentation

The official documentation –website, Guides, API documentation– should be updated not to use `String` prototype extensions.
This documentation should already use the new modules API from an effort to update it for Ember 2.16.

The Guides section on _disabling prototype extension_ will need to be updated when `String` prototype extensions are removed from Ember.

Resources owned by the Ember teams, such and Ember and Ember Data code bases, the Super Rentals repository, or the builds app for the website, will be updated accordingly.

## `Ember.String.htmlSafe` and `Ember.String.isHTMLSafe`

The move of `htmlSafe` and `isHTMLSafe` from `Ember.String` to `@ember/template` should be documented as part of the [ember-rfc176-data](https://github.com/ember-cli/ember-rfc176-data) and related codemods efforts, as that project is the source of truth for the mappings between the `Ember` global namespace and `@ember`-scoped modules.

## `Ember.String.loc` and `import { loc } from '@ember/string';`, `Ember.String` to `@ember/string`, `String` prototype extensions

An entry to the [Deprecation Guides](https://emberjs.com/deprecations/) will be added outlining the different recommended transition strategies.

### `Ember.String.loc`, `import { loc } from '@ember/string';`

As this function is deprecated, users will be recommended to use a [dedicated localization solution](https://emberobserver.com/categories/internationalization).

### `Ember.String` to `@ember/string`

The way that `@ember`-scoped modules will work in 2.16 is that `ember-cli-babel` will convert something like `import { dasherize } from '@ember/string';` to `import Ember from 'Ember'; const dasherize = Ember.String.dasherize;`.
What this means is that `import { dasherize } from '@ember/string';` will trigger a deprecation if you do not have the `@ember/string` package in your dependencies.

To address the above deprecation you will need to update `ember-cli-babel` to a a specific minor version or higher, to make sure it has the logic to detect `@ember/string`. The specific minor version will be known at the time the deprecation guide is written.
You will also need to add `@ember/string` to your application's development dependencies, or your addon's dependencies.

### `String` prototype extensions

If you are using `'myString'.dasherize()` or one of the other functions added to `String`, you will be instructed to replace that usage with `import { dasherize } from '@ember/string'; dasherize('myString')`, in addition to the changes on the previous section.

In case your code base is complicated enough that migrating all these usages at the same time is not convenient, you will be able to add `ember-string-prototype-extensions` to your dependencies, which will bring back extensions, without deprecations.

# Drawbacks

A lot of addons that deal with names depend on this behaviour, so they will need to install the addon. Also, Ember Data and some external serializers require these functions.

`htmlSafe` and `isHTMLSafe` would need to change packages, thus the reason to try and provide an Ember Watson recipe.

Another side-effect of this change is that certain users might be shipping duplicated code between `Ember.String` and `@ember/string`, but it is a necessary stepping stone and might be able to be addressed via svelting.

# Alternatives

Leave things as they are.

# Unresolved questions

None.
