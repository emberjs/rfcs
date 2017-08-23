- Start Date: 2017-07-14
- RFC PR: (leave this empty)
- Ember Issue: (leave this empty)

# Summary

This RFC proposes to deprecate the prototype extensions done by `Ember.String`, deprecate the `loc` method, and moving `htmlSafe` and `isHTMLSafe` to `@ember/component`.

# Motivation

Much of the public API of Ember was designed and published some time ago, when the client-side landscape looked much different. It was a time without without many utilities and methods that have been introduced to JavaScript since, without the current rich npm ecosystem, and without ES6 modules. On the Ember side, Ember CLI the subsequent addons were still to be introduced. Global mode was the way to go, and extending native prototypes like Ember does for `String`, `Array` and `Function` was a common practice.

With the introduction of [RFC #176](https://github.com/emberjs/rfcs/blob/master/text/0176-javascript-module-api.md), an opportunity to reduce the API that is shipped by default with an Ember application appears. A lot of nice-to-have functionality that was added at that time can now be moved to optional packages and addons, where they can be maintained and evolved without being tied to the core of the framework.

In the specific case of `Ember.String`, our goal is that users that need who these utility functions will include `@ember/string` in their dependencies, or rely on common utility packages like [`lodash.camelcase`](https://lodash.com/docs/#camelCase).

# Transition Path

It is important to understand that the transition path will be done in the context of the new modules API defined in RFC #176, which is scheduled to land in 2.16.
As this will likely be first of many packages to be extracted from the Ember source, the transition path arrived on needs to be clear and user-friendly.

## What is happening for framework developers?

The order of operations will be as follows:

* Move `htmlSafe` and `isHTMLSafe` to `@ember/component`
  * Update https://github.com/ember-cli/ember-rfc176-data
* Create an `@ember/string` package with the remaining public API
* Create an `ember-string-prototype-extensions` package that introduces `String` prototype extensions to aid in transitioning
* Make `ember-cli-babel` aware of the `@ember/string` package so it tells `babel-plugin-ember-modules-api-polyfill` not to convert those imports to the global `Ember` namespace
* Update usages in Ember and Ember Data codebases so that the projects do not trigger deprecations
* Deprecate `Ember.String`
  * Write deprecation guide which mentions minimum version of `ember-cli-babel`, and how/when to use `@ember/string` and `ember-string-prototype-extensions` packages
* Deprecate `loc` in `@ember/string`

## What is happening for framework users?

If you are using `Ember.String.loc`, you will be instructed to move to a dedicated localization solution, as this method will be completely deprecated.

If you are using `Ember.String.htmlSafe` or `Ember.String.isHTMLSafe`, you will be instructed to run the [`ember-modules-codemod`](https://github.com/ember-cli/ember-modules-codemod) and it will update to the correct imports from the `@ember/component` package.

If you are using one of the other `Ember.String` methods, like `Ember.String.dasherize`, you will receive a deprecation warning to inform you that you should run the [`ember-modules-codemod`](https://github.com/ember-cli/ember-modules-codemod), update `ember-cli-babel` to a specific minor version, and add `@ember/string` to your application's or addon's dependencies.

If you are using the `String` prototype extensions, like `"myString".dasherize()`, on top of the previous instructions you will be instructed to install `ember-string-prototype-extensions` in case updating the code to `dasherize("myString")` is not trivial.

## Timeline

* Deprecations are introduced - Ember 2.x
  * `String` protoype extensions are deprecated
  * `Ember.String` functions are deprecated
  * `loc` is completely deprecated
* Transition packages are introduced - Ember 2.x
  * `@ember/string`, which replaced `Ember.String`
  * `ember-string-prototype-extensions`, which brings `String` prototype extensions back
* Deprecations are removed - Ember 3.x, `@ember/string` 2.x
  * New major version of Ember is released
  * New major version of `@ember/string` is released

# How We Teach This

Ember guides' section on _disabling prototype extension_ would need to remove references to these methods. `camelize` is also used in the services tutorial.

Main usage of these functions are in blueprints or connecting to APIs where converting the type of case of keys was needed. It is also used in Ember Data.

For `htmlSafe` and `isHTMLSafe`, the move is easier since the methods are easier related to components than to strings.

In any case, a basic Ember Watson recipe would be easy to provide.

# Drawbacks

A lot of addons that deal with names depend on this behaviour, so they will need to install the addon. Also, Ember Data and some external serializers require these functions.

`htmlSafe` and `isHTMLSafe` would need to change packages, thus the reason to try and provide an Ember Watson recipe.

# Alternatives

Leave things as they are.

# Unresolved questions
