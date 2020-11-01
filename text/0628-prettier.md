---
Start Date: 2020-05-18
Relevant Team(s): Ember CLI
RFC PR: https://github.com/emberjs/rfcs/pull/628
Tracking: (leave this empty)

---

# Add Prettier

## Summary

This RFC proposes adding [Prettier](https://prettier.io/) to the blueprints
that back `ember new` and `ember addon`.

## Motivation

Using Prettier removes the ever present and ongoing stylistic debates that tend
to happen on teams. Prettier is incredibly freeing for both developers **and**
reviewers of a codebase. The developer can author in whatever format they find
easiest to type out, avoiding needless wasted time trying to get that
indentation _just_ right, and either setup an automated "format on save"
operation in their editor or use a quick command line invocation to `--fix` all
files at once.

This dove tails very nicely with Embers deeply ingrained goal of using shared
solutions to solve common problems, and aligns very well with the general trend
to play better with the general JavaScript ecosystem.

## Detailed design

Due to the pre-existing design (and improvements added to the blueprints in
[ember-cli/rfcs#121](https://github.com/emberjs/rfcs/blob/master/text/0121-remove-ember-cli-eslint.md))
implementation is very straight forward. The general idea is that we will
update the `app` and `addon` blueprints to add a few Prettier related packages
to the `package.json`, update the linting configuration to enforce Prettier
styles, and update the other blueprint code to ensure that its output is
Prettier compatible.

### Packages

The following packages will be added to the `package.json` of both `app` and `addon` blueprints:

* [prettier](https://www.npmjs.com/package/prettier) - The main package (used by the other packages).
* [eslint-config-prettier](https://www.npmjs.com/package/eslint-config-prettier) - Disables stylistic ESLint rules that would conflict with the Prettier output.
* [eslint-plugin-prettier](https://www.npmjs.com/package/eslint-plugin-prettier) - Enables `eslint-config-prettier`s `recommended` set, and adds the `prettier/prettier` rule to enforce styles.

### Configuration Changes

#### ESLint

The `.eslintrc.js` that is generated will be updated to add:

```js
{
  extends: ['plugin:prettier/recommended'],
}
```

#### Prettier

A `.prettierrc.js` will be added with the following contents:

```js
module.exports = {
  singleQuote: true,
};
```

There is a [small number of configuration
options](https://prettier.io/docs/en/options.html) that can be used in this
file, but we are intentionally avoiding modifying the default values for things
that are not _nearly_ universally agreed on in the Ember ecosystem.

A `.prettierignore` will be added to match the `.eslintignore` contents:

```
# unconventional js
/blueprints/*/files/
/vendor/

# compiled output
/dist/
/tmp/

# dependencies
/bower_components/
/node_modules/

# misc
/coverage/
!.*

# ember-try
/.node_modules.ember-try/
/bower.json.ember-try
/package.json.ember-try
```

#### Git

The `.gitignore` will be updated to add `.eslintcache` so that when using
`eslint --cache` the cache file itself won't be added to the repositories
history.

### Blueprint Changes

In general, it is recommended that all blueprints provided by addons should
satisfy the default linting configuration of a new Ember application. As such
the blueprints provided by `ember-cli`, `ember-source`, and `ember-data` will
be updated to ensure that they satisfy these new linting rules requirements.

In order to ensure blueprint output follows each individual projects custom
stylistic linting settings as much as possible. The following will be updated
to run `lint:fix` (when available) after their existing functionality:

* `ember generate`
* `ember init`
* `ember-cli-update`

#### `package.json` scripts

The `app` and `addon` blueprints will be updated to add the following
additional entries to `scripts`:

```json
{
  "scripts": {
    "lint": "npm-run-all --aggregate-output --continue-on-error --parallel 'lint:!(fix)'",
    "lint:fix": "npm-run-all --aggregate-output --continue-on-error --parallel lint:*:fix",
    "lint:js": "eslint . --cache",
    "lint:js:fix": "eslint . --fix",
    "lint:hbs:fix": "ember-template-lint . --fix"
  }
}
```

A few callouts here:

* The `lint:fix`, `lint:js:fix`, `lint:hbs:fix` scripts are new (introduced with this RFC)
* The `lint` script will be updated to ensure that it avoids running the new `lint:fix` script
* The `lint:js` script will be updated to add caching

This configuration is specifically intending to allow users to add additional
linters (e.g. `stylelint` or `markdownlint`) by adding scripts for them, and
they would automatically be rolled up into `lint:fix`.

### Codemod

In order to make the migration as simple as possible, a codemod will be created: `ember-cli-prettier-codemod`.

The codemod will do the following:

* Install the new/required dependencies
* Migrate an application / addon to the new linting configuration
* Commit _just_ the configuration changes.
* Run the new `lint:fix` script to update all files to the new format.
* Commit just the auto-fixed output.

While this seems quite simple (and in fact it is pretty easy), leveraging a
codemod will make it much easier to deal with rebasing large Prettier migration
pull requests. If one of those pull requests gets out of date (e.g. due to a
conflict in one of the files) you can simply re-run the codemod on the branch
and it will replace the previously created commits.

## How we teach this

We do not currently discuss linting or stylistic formatting in either guides.emberjs.com or cli.emberjs.com.

This RFC proposes adding a new subsection that discusses linting under [`Basic
Use` section of the CLI guides](https://cli.emberjs.com/release/basic-use/). In
addition, when this RFC is implemented we should do a throughough audit of the
`cli.emberjs.com` and `guides.emberjs.com` content to ensure that all code
snippets are formatted correctly (e.g. using the Prettier specific formatting
introduced here).

## Drawbacks

> The largest drawback is generally the cost of the _initial_ migration to Prettier. That migration tends to be quick, but it can cause pain for folks maintaining long lived branches.

## Alternatives

> Why not adopt [standard](https://standardjs.com/) instead of Prettier?

Prettier is **much** more popular with [9,249,847 weekly
downloads](https://www.npmjs.com/package/prettier) vs [265,658 weekly
downloads](https://www.npmjs.com/package/standard) (we all know these download
numbers don't mean a ton, but the order of magnitude can tell us something),
and (aside from the defaulting of `singleQuotes`) is much more aligned with the
Ember ecosystems inherent preferences.

Additionally, a very large number of the Ember ecosystem
projects are _already_ using Prettier internally. These include `ember-source`,
`ember-cli`, `ember-data`, `@ember/test-helpers`, `eslint-plugin-ember`, the
various packages composing the rendering engine, `@glimmer/component`,
`ember-template-lint`, and many more. Additionally a large number of community
maintained addons are also using it already ([here is a
listing](https://emberobserver.com/code-search?codeQuery=prettier&fileFilter=package.json&sort=score&sortAscending=false)
using EmberObservers awesome code search feature).

## Unresolved questions

> In general, users of `yarn` could be perfectly happy with `yarn lint:js --fix` / `yarn lint:hbs --fix` (which works today). Should we still add the `lint:js:fix`/`lint:hbs:fix` scripts?

I went with "yes" here in the initial RFC prose, as I'd general prefer to _reduce_ the differences between users of `npm` and `yarn` over time.
