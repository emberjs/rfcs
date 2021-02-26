---
Stage: Accepted
Start Date: 2021-01-09
Release Date: Unreleased
Release Versions:
  ember-cli: vX.Y.Z
Relevant Team(s): Ember CLI
RFC PR: https://github.com/emberjs/rfcs/pull/702
---

# Add eslint-plugin-qunit to ember-cli blueprint

## Summary

This RFC proposes adding [eslint-plugin-qunit](https://github.com/platinumazure/eslint-plugin-qunit) to the blueprints
that back `ember new` and `ember addon`.

## Motivation

Ember apps and addons already come with a number of linting plugins targeting various areas of the code:

* ember-template-lint - handlebars best practices
* eslint - general JavaScript best practices
* eslint-plugin-ember - Ember best practices
* eslint-plugin-node - Node best practices
* prettier - automatic styling (as a result of a recent [RFC](https://github.com/emberjs/rfcs/blob/master/text/0628-prettier.md))

But there's an important aspect of Ember apps that is not yet targeted by specialized linting: testing. Test code can easily make up half of the code in an Ember app, and well-written tests are of course critical to application quality.

[QUnit](https://qunitjs.com/) is the default testing framework used by Ember apps and the good news is that a popular and mature QUnit linting plugin is available for it: eslint-plugin-qunit. This plugin has been around for five years and is used in thousands of applications including many Ember applications. It has over 30 rules for enforcing best testing practices and detecting broken or incorrectly-written tests.

## Detailed design

The general idea is that we will update the `app` and `addon` blueprints to add the eslint-plugin-qunit package to the `package.json`, and update the linting configuration to extend the new presets.

### Packages

The following packages will be added to the `package.json` of both `app` and `addon` blueprints:

* [eslint-plugin-qunit](https://www.npmjs.com/package/eslint-plugin-qunit)

### Configuration Changes

The `.eslintrc.js` that is generated will be updated to extend the `qunit` linting configurations.

This change will be limited to test files only using an override, similar to how Node linting is limited to Node files only with an override. Using an override to scope linting to specific files provides additional protection against unwanted impacts (i.e. false positives or performance hits).

```js
{
  overrides: [
    ...,
    {
      files: ['tests/**/*'],
      extends: ['plugin:qunit/recommended', 'plugin:qunit/two'],
    },
  ],
}
```

## How we teach this

We do not currently discuss linting in either guides.emberjs.com or cli.emberjs.com.

Users will be able to find documentation for the new rules on the [eslint-plugin-qunit](https://github.com/platinumazure/eslint-plugin-qunit) GitHub page. Many IDEs will provide a link directly to the rule documentation on highlighted lint violations.

Any violations of the new rules will be detected by `yarn lint` and in some cases autofixed by `yarn lint:fix`.

## Drawbacks

For those introducing eslint-plugin-qunit to an existing codebase, the largest drawback is generally the initial cost of fixing linting violations. This can be mitigated by individually disabling noisy lint rules and working to fix violations overtime. But note that this RFC is more likely to impact newly-generated applications where there will be no existing lint violations to fix as opposed to existing applications.

In rare cases, users may be using an alternative testing framework like Mocha or Jest, and they can safely ignore or remove eslint-plugin-qunit.

## Alternatives

The implementation is straightforward and there are no known alternative implementations for adding more test-oriented linting.
