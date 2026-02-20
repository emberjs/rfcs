---
stage: accepted
start-date: 2023-01-07T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
prs:
  accepted: https://github.com/emberjs/rfcs/pull/894
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

# Add eslint-plugin-import to ember-cli blueprint

## Summary

This RFC proposes adding [eslint-plugin-import](https://github.com/import-js/eslint-plugin-import) to the blueprints
that back `ember new` and `ember addon`.

## Motivation

Ember apps and addons already come with a number of linting plugins targeting various areas of the code:

* [ember-template-lint](https://github.com/ember-template-lint/ember-template-lint) - Handlebars / templating best practices
* [eslint](https://eslint.org/) - General JavaScript best practices
* [eslint-plugin-ember](https://github.com/ember-cli/eslint-plugin-ember) - Ember best practices
* [eslint-plugin-node](https://github.com/mysticatea/eslint-plugin-node) - Node best practices
* [eslint-plugin-qunit](https://github.com/platinumazure/eslint-plugin-qunit) - Testing best practices ([RFC](https://github.com/emberjs/rfcs/blob/master/text/0702-eslint-plugin-qunit.md))
* [prettier](https://prettier.io/) - Code styling ([RFC](https://github.com/emberjs/rfcs/blob/master/text/0628-prettier.md))
* [stylelint](https://stylelint.io/) - CSS styling ([RFC](https://github.com/emberjs/rfcs/blob/master/text/0814-stylelint.md))

But there's an important aspect of Ember apps that is not yet targeted by specialized linting: import statements. Imports are present in almost every file, and various problems or even crashes can result from invalid imports.

[eslint-plugin-import](https://github.com/import-js/eslint-plugin-import) is one of the top few, most popular ESLint plugins, with around 8 million dependents on GitHub, 4.5k stars on GitHub, and 17 million weekly downloads on NPM, as of this writing. It contains 40+ rules for detecting problems with imports as well as enforcing usage preferences.

## Detailed design

The general idea is that we will update the `app` and `addon` blueprints to add the eslint-plugin-import package to the `package.json`, and update the linting configuration to extend the relevant configurations.

### Packages

The following packages will be added to the `package.json` of both `app` and `addon` blueprints:

* [eslint-plugin-import](https://www.npmjs.com/package/eslint-plugin-import)

### Configuration changes

The ESLint config file that is generated will be updated to extend the `recommended` linting configuration:

```js
// .eslintrc.js
{
  extends: ['plugin:import/recommended'],
}
```

We can also configure the [ember-cli-typescript](https://github.com/typed-ember/ember-cli-typescript) blueprint to enable the `typescript` config, but only for TypeScript files:

```js
// .eslintrc.js
{
  overrides: [
    {
      files: ['*.ts'],
      extends: ['plugin:import/typescript'],
    },
  ],
}
```

Note that these configurations enable only a handful of key rules from the plugin's 40+ rules. We may optionally choose to individually enable additional rules that we determine to be particularly valuable for Ember apps, either in this RFC, or in the future. But regardless, getting the plugin with its basic linting setup into place right now is worthwhile on its own, as this will also make it much easier for users to enable rules on their own too.

### Resolver

We will disable the [import/no-unresolved](https://github.com/import-js/eslint-plugin-import/blob/HEAD/docs/rules/no-unresolved.md) rule due to the potential infeasibility of reliably resolving imports in current Ember apps.

```js
// .eslintrc.js
{
  rules: {
    'import/no-unresolved': 'off',
  },
}
```

The difficulties involved are described in [this](https://github.com/emberjs/rfcs/pull/894#issuecomment-1450702826) comment, and also demonstrated by various false positives from the rule that can be seen below with imports from Ember packages, as well as import paths prefixed with the current Ember app name, such as:

```js
import { service } from '@ember/service';
// Triggers violation: Unable to resolve path to module '@ember/service'

import Ember from 'ember';
// Triggers violation: Unable to resolve path to module 'ember'

// app/app.js
import config from 'my-app-name/config/environment';
// Triggers violation: Unable to resolve path to module 'my-app-name/config/environment'
```

There's an old resolver [eslint-import-resolver-ember](https://github.com/gabrielcsapo/eslint-import-resolver-ember) which did not appear to help during my testing and may have only been helpful in limited scenarios. Any future resolver would require significant modernization and testing to ensure it works with TypeScript, different module types, different app/addon/engine structures, different Ember versions, etc. If such a resolver becomes practical in the future, it could be published from the [ember-cli](https://github.com/ember-cli) organization.

## How we teach this

We do not currently discuss linting in either [guides.emberjs.com](https://guides.emberjs.com/) or [cli.emberjs.com](https://cli.emberjs.com/).

Users will be able to find documentation for the new rules on the [eslint-plugin-import](https://github.com/import-js/eslint-plugin-import) GitHub page. Many IDEs will provide a link directly to the rule documentation on highlighted lint violations.

Any violations of the new rules will be detected by our existing `npm run lint:js` script and in some cases autofixed by `npm run lint:js:fix`.

## Drawbacks

For those introducing eslint-plugin-import to an existing codebase, the largest drawback is generally the initial cost of fixing linting violations. This can be mitigated by individually disabling noisy lint rules and working to fix violations over time. But note that this RFC is more likely to impact newly-generated applications where there will be no existing lint violations to fix, as opposed to existing applications.

Users not interested in linting imports can safely disable or remove eslint-plugin-import.

## Alternatives

The implementation is straightforward.

The main alternative is to do nothing, and leave it up to users to add eslint-plugin-import on their own. But if we are able to install and correctly setup this plugin for users, it will enable many more users to take advantage of this additional type of linting.

## Unresolved questions

* We will bypass the [resolver](#resolver) issues by disabling the rule.
* Ensure this is tested with TypeScript, different module types, different app/addon/engine structures, different Ember versions, etc.

