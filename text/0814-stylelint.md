---
Stage: Accepted
Start Date: 2022-04-19
Release Date: Unreleased
Release Versions:
ember-cli: vX.Y.Z
Relevant Team(s): Ember CLI
RFC PR: https://github.com/emberjs/rfcs/pull/814
---

# Add stylelint to Ember blueprints

## Summary

This RFC proposes adding [stylelint](https://stylelint.io/) to the blueprints 
that back `ember new` and `ember addon`.

## Motivation

Ember app and addons already come with a broad variety of linting plugins targeting various areas 
of our codebases:

* [ember-template-lint](https://github.com/ember-template-lint/ember-template-lint) - Handlebars & Ember Templating Best Practices
* [eslint](https://eslint.org/) - General JavaScript Best Practices
* [eslint-plugin-ember](https://github.com/ember-cli/eslint-plugin-ember) - Ember JavaScript Best Practices
* [eslint-plugin-node](https://github.com/mysticatea/eslint-plugin-node) - Node Best Practices
* [eslint-plugin-qunit](https://github.com/platinumazure/eslint-plugin-qunit) - QUnit Best Practices [RFC](https://github.com/emberjs/rfcs/blob/master/text/0702-eslint-plugin-qunit.md)
* [prettier](https://prettier.io/) - Automated Code Styling [RFC](https://github.com/emberjs/rfcs/blob/master/text/0628-prettier.md)

With all these amazing tools in place, however, we still have a major aspect of our Ember codebases 
that are not benefiting from mature tooling to enforce best practices and automate code styling: CSS.

[Stylelint](https://github.com/stylelint/stylelint) is a modern linter for stylesheets that ships with 
over 170 built-in rules, sane default configs, and robust plugin support.

## Detailed design

The general idea is that we will update the `app` and `addon` [blueprints](https://github.com/ember-cli/ember-cli/tree/master/blueprints) 
to add a few Stylelint-related packages to the `package.json`, add config files to enable this tooling 
by default, and add commands to the `scripts` section of `package.json` to expose easy access to this 
tool consistent with our existing linters.

### Packages

The following packages will be added to the `devDependencies` section of both blueprints:

* [stylelint](https://github.com/stylelint/stylelint) - The base linter
* [stylelint-config-standard](https://github.com/stylelint/stylelint-config-standard) - The standard rule config supported by the stylelint community
* [stylelint-prettier](https://github.com/prettier/stylelint-prettier) - Plugin to support prettier auto-formatting as stylelint rules
* [stylelint-config-prettier](https://github.com/prettier/stylelint-config-prettier) - Plugin to disable stylelint rules that conflict with prettier's automatic behavior

Prettier itself is already included in the Ember blueprints by default.

### Configuration Changes

A `.stylelintrc.js` file will be added with the following contents:

```js title=".stylelintrc.js"
'use strict';

module.exports = {
  extends: ['stylelint-config-standard', 'stylelint-prettier/recommended'],
};
```

There are many rules, configs, and plugins available for stylelint and prettier, but we are 
intentionally avoiding being prescriptive on anything that isn't nearly universally agreed upon 
by the stylelint, prettier, and Ember communities.

### Blueprint Changes

In general, it is recommended that all blueprints provided by addons should satisfy the default 
linting configuration of a new Ember application. As such, the blueprints provided by ember-cli 
will be updated as needed to ensure that they satisfy these new linting requirements.

#### `package.json` scripts

The `app` and `addon` blueprints will be updated to add style-related linting scripts:

```json title="package.json"
{
  "scripts": {
    "lint:css": "stylelint \"**/*.{css,scss,sass,less}\"",
    "lint:css:fix": "stylelint \"**/*.{css,scss,sass,less}\" --fix",
  }
}
```

#### Linting Stylelint's Config

`.stylelintrc.js` should be [linted as a node file](https://github.com/ember-cli/ember-cli/blob/master/blueprints/app/files/.eslintrc.js#L26-L38) 
in eslint's config.

#### npmignore

`/.stylelintrc.js` should be added to the `addon` blueprint's [`npmignore` file](https://github.com/ember-cli/ember-cli/blob/master/blueprints/addon/files/npmignore).

## How we teach this

We do not currently discuss linting or stylistic formatting in either guides.emberjs.com or cli.emberjs.com.

However, [previous linting RFCs](https://github.com/emberjs/rfcs/blob/master/text/0628-prettier.md) have 
proposed adding a subsection to the CLI guides to discuss linting. As that is implemented, we can include 
basic information about style linting as well.

Documentation for the new rules can be found within the Doc sites & READMEs for the packages being added by this RFC.

## Drawbacks

For those introducing stylelint to an existing codebase, the largest drawback is generally the 
initial cost of fixing linting violations. This can be mitigated by individually disabling noisy 
lint rules and working to fix violations overtime, or by extending the [lint-todo](https://github.com/lint-todo/) 
functionality available for `ember-template-lint` and `eslint` to support `stylelint` as well. 
However, note that this RFC is more likely to impact newly-generated applications where there 
will be no existing lint violations to fix as opposed to existing applications.

## Alternatives

>Ember applications may choose to use CSS, SCSS, LESS, or other approaches for styling. How do we 
>reconcile our default linting config?

Today, the de facto default for Ember apps via the blueprints is CSS, so our default configuration 
should strive to support that use-case. `stylelint` has readily-available support for popular 
preprocessors built-in, and there are plugins like [`stylelint-scss`](https://github.com/stylelint-scss/stylelint-scss) 
that add rules specific to those ecosystems for teams to opt into.

>Why name the scripts `lint:css` instead of something more universal like `lint:style`?

With CSS being the default in Ember applications, this is consistent with our existing linting strategy. 
While teams may choose to adopt a preprocessor like SASS in the same way they could adopt TypeScript 
over JavaScript, we still default to `lint:js` and `lint:hbs` to match the standard file types of 
a new Ember application.

>Why use `stylelint-config-standard` over `stylelint-config-recommended`?

The standard config intentional extends the recommended config, which is just a smaller subset of rules. 
The [README](https://github.com/stylelint/stylelint-config-standard/blob/main/README.md) for the standard 
config talks briefly about the philosophy behind the additional rules it adds. If there is a strong case 
for the standard config being a divergence from how the Ember community would commonly author stylesheets, 
we could fall back to the recommended config instead. However, the standard config appears to provide sane 
defaults for new applications.

## Unresolved questions

This RFC will be updated as such questions are posed, until we have an answer.