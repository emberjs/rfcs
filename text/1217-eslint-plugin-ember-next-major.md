---
stage: accepted
start-date: 2026-07-27T00:00:00.000Z # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - framework
  - learning
  - steering
prs:
  accepted: https://github.com/emberjs/rfcs/pull/1217
project-link:
suite:
---

# The next major of eslint-plugin-ember

## Summary

This RFC defines the `recommended` config for the next major of `eslint-plugin-ember` (v14).

- The template rules that `ember-template-lint` enabled by default become enabled by default for gjs/gts files. This covers the rules that apply to strict mode. [RFC #1214 "Deprecate ember-template-lint"][rfc-1214] committed us to this config change.
- `recommended` stays gjs/gts only. Linting of `.hbs` files stays opt-in through the `hbs` config, which keeps the full `ember-template-lint` parity set.
- Rules that catch patterns from `ember-source` 3.x and earlier come out of `recommended`.
- A number of other recommended rule changes (see appendices)
- The `recommended-gjs` and `recommended-gts` configs are removed. `recommended` now carries their rules, scoped per file type.

[eslint-plugin-ember#2158][issue-2158] requires an RFC for each change to a recommended rule set. This is that RFC. [eslint-plugin-ember#2060][issue-2060] tracks the plan for the release itself.

[rfc-1214]: https://github.com/emberjs/rfcs/pull/1214
[issue-2158]: https://github.com/ember-cli/eslint-plugin-ember/issues/2158
[issue-2060]: https://github.com/ember-cli/eslint-plugin-ember/issues/2060

## Motivation

This major has three motivations.

1. [RFC #1214][rfc-1214] deprecates `ember-template-lint` and unifies all Ember lint rules in `eslint-plugin-ember`. `eslint-plugin-ember@13` provides every `ember-template-lint` rule as an `ember/template-*` rule[^no-partial]. None of those rules are in `recommended` yet. They stay opt-in through the `template-lint-migration` config, so that a project with both tools does not get two errors for each violation.

   `ember-template-lint` is now deprecated, so the template rules must be on by default. If they are not, a new app loses lint coverage that every Ember app has had for years. That coverage includes the A11y rules, and the Ember project has a core commitment to those rules on by default.

2. The current `recommended` set spends lint time on patterns that cannot exist in an app on `ember-source` 4+. Some of those rules are also the most expensive rules in the set. Profiling in [eslint-plugin-ember#2060][issue-2060] measured the two most expensive at about 6.2 seconds _each_ on a large app. Each of the two is 5.9% of the total lint time. Both check for API that ember-source removed years ago.

3. A major release is the only place where `recommended` can gain a rule that reports on existing code. Two such rules are ready, and only the semver cost holds them back. Both catch patterns that an app on the latest `ember-source` can still reach. That is the opposite of the rules that motivation 2 removes.

[^no-partial]: every rule except the one for `{{partial}}`. ember-source removed `{{partial}}` in 4.0, so no code is left to lint.

## Detailed design

### Add the strict-mode template rules to `recommended`

The rules in the [`template-lint-migration` config][migration-config] that apply to strict-mode templates go into `recommended`, scoped to `**/*.{gjs,gts}`. [Appendix A](#appendix-a-template-rules-added-to-recommended) has the full list. The baseline is the `recommended` preset of `ember-template-lint`, plus one rule that converts each leftover `{{! template-lint-disable ... }}` comment to an eslint directive through `eslint --fix`.

Two groups of rules do not apply to strict-mode templates. Both stay in the hbs config only ([Appendix B](#appendix-b-loose-mode-only-rules-not-added-to-recommended)):

- Rules that prepare a loose-mode template for strict mode. A gjs/gts file is already there, and the recommended preset of `ember-template-lint` also disables these rules for gjs/gts.
- Rules for constructs that strict mode cannot express, such as `{{action}}`, curly `{{input}}`, and `{{#with}}`. These rules can never report in a gjs/gts file, so they only cost lint time.

`recommended` today covers gjs/gts and js/ts only, and that does not change. It never reads `.hbs` files, so a project can add it without an hbs parser. A new app has no `.hbs` files, so `recommended` alone gives that app full lint parity with `ember-template-lint`.

[migration-config]: https://github.com/ember-cli/eslint-plugin-ember/blob/main/lib/config/template-lint-migration.js

> [!NOTE]
> The recommended preset of `ember-template-lint` had to disable its builtin-component rules for gjs/gts, because it has no knowledge of imports. The eslint versions see the whole module. They can tell whether `<Input>` is the one from `@ember/component`, or a local component with the same name. Those rules _do_ move to `recommended`. This is one of the motivations of [RFC #1214][rfc-1214].

### Add two JS rules to `recommended`

Two JS rules go into `recommended`. Both are already in the plugin as opt-in. Neither one is a template rule, so `recommended` scopes both the way it scopes its other JS rules, not to `**/*.{gjs,gts}`. [Appendix C](#appendix-c-js-rules-in-recommended) lists every JS rule in the set, and gives the reason for each new one.

### Remove the `recommended-gjs` and `recommended-gts` configs

`recommended` now carries the gjs and gts rules itself, in blocks that `files` scopes. Both extra configs are redundant, so v14 removes them.

`recommended-gts` did one more thing. It turned off the core rules that TypeScript already reports, and it turned on the four that TypeScript makes better. That set comes from the `eslint-recommended` config of typescript-eslint. `recommended` now applies it to `**/*.gts`.

The result is one config in place of three:

```js
// eslint.config.mjs
import emberRecommended from 'eslint-plugin-ember/configs/recommended';

export default [
  ...emberRecommended,
  // a parser block per file type is still required
];
```

An app that extends `plugin:ember/recommended-gjs` or `plugin:ember/recommended-gts` deletes those two lines. `recommended` sets every rule that they set.

### Linting `.hbs` files stays opt-in

An app with loose-mode templates opts in with the `hbs` config, which v14 renames from [`template-lint-migration`][migration-config]. It keeps the _full_ `ember-template-lint` parity set, and that set includes the loose-mode-only rules that stay out of `recommended`. The app also needs a parser block that routes `.hbs` files to the hbs parser. `eslint-plugin-ember` already ships that parser as `ember-eslint-parser/hbs`.

```js
// eslint.config.mjs
import { hbsParser, plugin as ember } from 'eslint-plugin-ember/recommended';
import emberHbs from 'eslint-plugin-ember/configs/hbs';

export default [
  // ... existing config ...
  {
    files: ['app/**/*.hbs'],
    plugins: { ember },
    languageOptions: { parser: hbsParser },
  },
  ...emberHbs,
];
```

> [!IMPORTANT]
> If a config has a `@typescript-eslint/parser` block with a broad `files` glob, narrow that glob to `['**/*.{js,ts,gjs,gts}']`. Flat config merges every block that matches a file, and a type-aware rule reports an error on a file that is not JS.

The migration guide that [RFC #1214][rfc-1214] requires documents this config block. The block does not go in the app blueprint, because a new app has no `.hbs` files.

### Remove legacy rules from `recommended`

[Appendix D](#appendix-d-rules-removed-from-recommended) lists the rules that come out of `recommended`. Each one lints for API that `ember-source` 4+ does not have. No app that can install `eslint-plugin-ember@14` can trigger them, so they only cost lint time.

The plugin keeps every one of these rules. They only come out of `recommended`. An app that still supports an older ember-source can turn them on again in its own config:

```js
export default [
  ...emberRecommended,
  {
    rules: {
      // whichever of the removed rules still apply
    },
  },
];
```

That app can also stay on `eslint-plugin-ember@13` until it is on ember-source 4+.

### What happens to `template-lint-migration`

The config stays for as long as `.hbs` exists. An app with `.hbs` files uses it, and hbs coverage does not change in this major. The migration guide from [RFC #1214][rfc-1214] is written around it.

This config is no longer a migration config, so v14 renames it to `hbs`. The old name stops working in v14, and the release notes carry the rename.

### Other breaking changes

[#2060][issue-2060] also tracks routine breaking changes for the release. The plugin drops EOL node versions, matches the supported version range of eslint, and cleans up its config exports. Those changes do not change what the plugin lints, so they need no RFC and are not specified here. This RFC covers the `recommended` rule set.

The two config removals above are the exception. This RFC specifies them, because the typescript-eslint disables move into `recommended` and change what the plugin lints in a gts file.

## How we teach this

[RFC #1214][rfc-1214] already requires most of the teaching work. Its migration guide from `ember-template-lint` covers the `.hbs` parser block and the directive conversion (`template-lint-disable` → `eslint-disable-next-line`). The guide also covers the move of custom rule overrides from `.template-lintrc.js` into `eslint.config.mjs`.

For the v14 upgrade itself:

- A new app gets the new config from the blueprint. It needs no teaching.
- An existing app sees new errors in its gjs/gts files after the upgrade. The two added JS rules also report new errors in js/ts files. The release notes point at [eslint bulk suppressions](https://eslint.org/blog/2025/04/introducing-bulk-suppressions/) and [Lint to the Future](https://github.com/mansona/lint-to-the-future). Both let a team adopt the new rules one part at a time, in place of one large PR. This replaces the `lint-todo` workflow of `ember-template-lint`, per RFC #1214.
- An app with `.hbs` files opts in to hbs linting with the hbs config. That is a task for the migration from `ember-template-lint`, which the RFC #1214 guide covers. It is not a v14 upgrade task.
- An app that already uses the `template-lint-migration` config sees no new template errors. For that app, v14 is a no-op.

- Ensure the Guides and Tutorial follow the new recommended config

## Drawbacks

- Not all rules have autofixes, but those that don't should already be linking to sufficient documentation to help developers have actionable information

- No `hbs` support by default (some have not yet migrated to gjs/gts)

## Alternatives

### Keep the template rules opt-in

This contradicts RFC #1214. A new app then ships without the default lint coverage that every Ember app has had for years, and that coverage includes A11y.

### Enable only the A11y subset by default

This keeps the A11y commitment and creates fewer new errors. It also breaks parity with what an `ember-template-lint` user has today, and it makes the question "did I migrate correctly?" harder to answer. Parity is easier to explain: the same rules, one tool.

### Ship a `legacy` config with the removed rules

[#1950][issue-1950] proposed this. It is one more config to name, to document, and to maintain, for a set of apps that gets smaller every year. Those apps already have two options. They can stay on v13, or they can turn on the few rules they still want in their own config. The cost of maintenance is too high for that result.

### Keep the two new JS rules opt-in

A rule that is already in `recommended` bans one import for the same pattern as one of the two new rules. That ban is not enough, because hand-written code reaches the same pattern without that import. [Appendix C](#appendix-c-js-rules-in-recommended) has the detail.

### Merge the template rules into `recommended` unscoped (including `.hbs`)

`recommended` then tries to lint `.hbs` files. An app without an hbs parser gets a parse error for each one. A gjs/gts-only `recommended` works with no parser setup, and hbs linting stays opt-in.

## Unresolved questions

n/a

## Appendix A: template rules added to `recommended`

87 rules, at `error`, scoped to `**/*.{gjs,gts}`. This is the `template-lint-migration` rule set without the loose-mode-only rules of Appendix B. That set is the recommended preset of `ember-template-lint`, minus `no-partial`, plus `template-no-template-lint-directives`. `no-partial` has no equivalent, because `{{partial}}` no longer exists.

`recommended` also enables [`ember/template-no-let-reference`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/template-no-let-reference.md) for gjs/gts today, and that does not change. `ember-template-lint` has no equivalent rule. The gjs/gts set is 88 rules in total.

- [`ember/template-builtin-component-arguments`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-builtin-component-arguments.md)
- [`ember/template-link-href-attributes`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-link-href-attributes.md)
- [`ember/template-link-rel-noopener`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-link-rel-noopener.md)
- [`ember/template-no-abstract-roles`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-abstract-roles.md)
- [`ember/template-no-accesskey-attribute`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-accesskey-attribute.md)
- [`ember/template-no-action-on-submit-button`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-action-on-submit-button.md)
- [`ember/template-no-args-paths`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-args-paths.md)
- [`ember/template-no-arguments-for-html-elements`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-arguments-for-html-elements.md)
- [`ember/template-no-aria-hidden-body`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-aria-hidden-body.md)
- [`ember/template-no-aria-unsupported-elements`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-aria-unsupported-elements.md)
- [`ember/template-no-array-prototype-extensions`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-array-prototype-extensions.md)
- [`ember/template-no-at-ember-render-modifiers`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-at-ember-render-modifiers.md)
- [`ember/template-no-attrs-in-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-attrs-in-components.md)
- [`ember/template-no-autofocus-attribute`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-autofocus-attribute.md)
- [`ember/template-no-block-params-for-html-elements`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-block-params-for-html-elements.md)
- [`ember/template-no-builtin-form-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-builtin-form-components.md)
- [`ember/template-no-capital-arguments`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-capital-arguments.md)
- [`ember/template-no-class-bindings`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-class-bindings.md)
- [`ember/template-no-debugger`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-debugger.md)
- [`ember/template-no-duplicate-attributes`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-duplicate-attributes.md)
- [`ember/template-no-duplicate-id`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-duplicate-id.md)
- [`ember/template-no-duplicate-landmark-elements`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-duplicate-landmark-elements.md)
- [`ember/template-no-empty-headings`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-empty-headings.md)
- [`ember/template-no-extra-mut-helper-argument`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-extra-mut-helper-argument.md)
- [`ember/template-no-forbidden-elements`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-forbidden-elements.md)
- [`ember/template-no-heading-inside-button`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-heading-inside-button.md)
- [`ember/template-no-html-comments`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-html-comments.md)
- [`ember/template-no-index-component-invocation`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-index-component-invocation.md)
- [`ember/template-no-inline-styles`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-inline-styles.md)
- [`ember/template-no-invalid-aria-attributes`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-invalid-aria-attributes.md)
- [`ember/template-no-invalid-interactive`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-invalid-interactive.md)
- [`ember/template-no-invalid-link-text`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-invalid-link-text.md)
- [`ember/template-no-invalid-link-title`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-invalid-link-title.md)
- [`ember/template-no-invalid-meta`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-invalid-meta.md)
- [`ember/template-no-invalid-role`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-invalid-role.md)
- [`ember/template-no-link-to-positional-params`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-link-to-positional-params.md)
- [`ember/template-no-link-to-tagname`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-link-to-tagname.md)
- [`ember/template-no-log`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-log.md)
- [`ember/template-no-negated-condition`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-negated-condition.md)
- [`ember/template-no-nested-interactive`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-nested-interactive.md)
- [`ember/template-no-nested-landmark`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-nested-landmark.md)
- [`ember/template-no-nested-splattributes`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-nested-splattributes.md)
- [`ember/template-no-obscure-array-access`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-obscure-array-access.md)
- [`ember/template-no-obsolete-elements`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-obsolete-elements.md)
- [`ember/template-no-outlet-outside-routes`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-outlet-outside-routes.md)
- [`ember/template-no-passed-in-event-handlers`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-passed-in-event-handlers.md)
- [`ember/template-no-pointer-down-event-binding`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-pointer-down-event-binding.md)
- [`ember/template-no-positional-data-test-selectors`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-positional-data-test-selectors.md)
- [`ember/template-no-positive-tabindex`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-positive-tabindex.md)
- [`ember/template-no-potential-path-strings`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-potential-path-strings.md)
- [`ember/template-no-quoteless-attributes`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-quoteless-attributes.md)
- [`ember/template-no-redundant-fn`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-redundant-fn.md)
- [`ember/template-no-redundant-role`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-redundant-role.md)
- [`ember/template-no-scope-outside-table-headings`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-scope-outside-table-headings.md)
- [`ember/template-no-shadowed-elements`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-shadowed-elements.md)
- [`ember/template-no-template-lint-directives`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-template-lint-directives.md)
- [`ember/template-no-triple-curlies`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-triple-curlies.md)
- [`ember/template-no-unbalanced-curlies`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-unbalanced-curlies.md)
- [`ember/template-no-unbound`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-unbound.md)
- [`ember/template-no-unknown-arguments-for-builtin-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-unknown-arguments-for-builtin-components.md)
- [`ember/template-no-unnecessary-component-helper`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-unnecessary-component-helper.md)
- [`ember/template-no-unnecessary-curly-parens`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-unnecessary-curly-parens.md)
- [`ember/template-no-unnecessary-curly-strings`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-unnecessary-curly-strings.md)
- [`ember/template-no-unsupported-role-attributes`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-unsupported-role-attributes.md)
- [`ember/template-no-unused-block-params`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-unused-block-params.md)
- [`ember/template-no-valueless-arguments`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-valueless-arguments.md)
- [`ember/template-no-whitespace-for-layout`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-whitespace-for-layout.md)
- [`ember/template-no-whitespace-within-word`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-whitespace-within-word.md)
- [`ember/template-no-yield-only`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-yield-only.md)
- [`ember/template-no-yield-to-default`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-yield-to-default.md)
- [`ember/template-require-aria-activedescendant-tabindex`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-aria-activedescendant-tabindex.md)
- [`ember/template-require-button-type`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-button-type.md)
- [`ember/template-require-context-role`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-context-role.md)
- [`ember/template-require-has-block-helper`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-has-block-helper.md)
- [`ember/template-require-iframe-title`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-iframe-title.md)
- [`ember/template-require-input-label`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-input-label.md)
- [`ember/template-require-lang-attribute`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-lang-attribute.md)
- [`ember/template-require-mandatory-role-attributes`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-mandatory-role-attributes.md)
- [`ember/template-require-media-caption`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-media-caption.md)
- [`ember/template-require-presentational-children`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-presentational-children.md)
- [`ember/template-require-valid-alt-text`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-valid-alt-text.md)
- [`ember/template-require-valid-named-block-naming-format`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-require-valid-named-block-naming-format.md)
- [`ember/template-simple-modifiers`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-simple-modifiers.md)
- [`ember/template-simple-unless`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-simple-unless.md)
- [`ember/template-splat-attributes-only`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-splat-attributes-only.md)
- [`ember/template-style-concatenation`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-style-concatenation.md)
- [`ember/template-table-groups`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-table-groups.md)

## Appendix B: loose-mode-only rules, not added to `recommended`

These 9 rules stay in the `hbs` config. They keep full `ember-template-lint` parity for `.hbs` files.

| Rule | Why it does not apply to gjs/gts |
| ---- | ------------------------------- |
| [`ember/template-no-implicit-this`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-implicit-this.md) | strict mode has no implicit `this`. An unresolved reference is a build error |
| [`ember/template-no-curly-component-invocation`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-curly-component-invocation.md) | strict mode has no component-or-property ambiguity |
| [`ember/template-no-action`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-action.md) | `{{action}}` has no strict-mode import |
| [`ember/template-no-route-action`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-route-action.md) | `{{route-action}}` has no strict-mode import |
| [`ember/template-no-input-block`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-input-block.md) | curly `{{input}}` cannot resolve in strict mode |
| [`ember/template-no-input-tagname`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-input-tagname.md) | curly `{{input}}` cannot resolve in strict mode |
| [`ember/template-no-with`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-with.md) | ember-source removed `{{#with}}` in 4.0, and strict mode cannot express it |
| [`ember/template-deprecated-render-helper`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-deprecated-render-helper.md) | `{{render}}` is long gone, and strict mode cannot express it |
| [`ember/template-deprecated-inline-view-helper`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-deprecated-inline-view-helper.md) | `{{view}}` is long gone, and strict mode cannot express it |

## Appendix C: JS rules in `recommended`

59 rules, at `error`, for js/ts and gjs/gts. Two of them are new in v14.

- [`ember/avoid-leaking-state-in-ember-objects`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/avoid-leaking-state-in-ember-objects.md)
- [`ember/classic-decorator-hooks`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/classic-decorator-hooks.md)
- [`ember/classic-decorator-no-classic-methods`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/classic-decorator-no-classic-methods.md)
- [`ember/jquery-ember-run`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/jquery-ember-run.md)
- [`ember/new-module-imports`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/new-module-imports.md)
- [`ember/no-actions-hash`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-actions-hash.md)
- [`ember/no-arrow-function-computed-properties`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-arrow-function-computed-properties.md)
- [`ember/no-assignment-of-untracked-properties-used-in-tracking-contexts`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-assignment-of-untracked-properties-used-in-tracking-contexts.md)
- [`ember/no-at-ember-render-modifiers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-at-ember-render-modifiers.md)
- [`ember/no-attrs-in-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-attrs-in-components.md)
- [`ember/no-attrs-snapshot`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-attrs-snapshot.md)
- [`ember/no-builtin-form-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-builtin-form-components.md) (new in v14)
- [`ember/no-capital-letters-in-routes`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-capital-letters-in-routes.md)
- [`ember/no-classic-classes`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-classic-classes.md)
- [`ember/no-classic-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-classic-components.md)
- [`ember/no-component-lifecycle-hooks`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-component-lifecycle-hooks.md)
- [`ember/no-computed-properties-in-native-classes`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-computed-properties-in-native-classes.md)
- [`ember/no-controller-access-in-routes`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-controller-access-in-routes.md)
- [`ember/no-duplicate-dependent-keys`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-duplicate-dependent-keys.md)
- [`ember/no-ember-super-in-es-classes`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-ember-super-in-es-classes.md)
- [`ember/no-ember-testing-in-module-scope`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-ember-testing-in-module-scope.md)
- [`ember/no-empty-glimmer-component-classes`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-empty-glimmer-component-classes.md)
- [`ember/no-get`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-get.md)
- [`ember/no-global-jquery`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-global-jquery.md)
- [`ember/no-incorrect-calls-with-inline-anonymous-functions`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-incorrect-calls-with-inline-anonymous-functions.md)
- [`ember/no-incorrect-computed-macros`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-incorrect-computed-macros.md)
- [`ember/no-invalid-debug-function-arguments`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-invalid-debug-function-arguments.md)
- [`ember/no-invalid-dependent-keys`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-invalid-dependent-keys.md)
- [`ember/no-invalid-test-waiters`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-invalid-test-waiters.md)
- [`ember/no-jquery`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-jquery.md)
- [`ember/no-legacy-test-waiters`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-legacy-test-waiters.md)
- [`ember/no-mixins`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-mixins.md)
- [`ember/no-modifier-without-element-usage`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-modifier-without-element-usage.md) (new in v14)
- [`ember/no-new-mixins`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-new-mixins.md)
- [`ember/no-noop-setup-on-error-in-before`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-noop-setup-on-error-in-before.md)
- [`ember/no-observers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-observers.md)
- [`ember/no-on-calls-in-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-on-calls-in-components.md)
- [`ember/no-pause-test`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-pause-test.md)
- [`ember/no-private-routing-service`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-private-routing-service.md)
- [`ember/no-restricted-resolver-tests`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-restricted-resolver-tests.md)
- [`ember/no-runloop`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-runloop.md)
- [`ember/no-settled-after-test-helper`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-settled-after-test-helper.md)
- [`ember/no-shadow-route-definition`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-shadow-route-definition.md)
- [`ember/no-side-effects`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-side-effects.md)
- [`ember/no-test-and-then`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-test-and-then.md)
- [`ember/no-test-import-export`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-test-import-export.md)
- [`ember/no-test-support-import`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-test-support-import.md)
- [`ember/no-test-this-render`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-test-this-render.md)
- [`ember/no-tracked-properties-from-args`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-tracked-properties-from-args.md)
- [`ember/no-unnecessary-route-path-option`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-unnecessary-route-path-option.md)
- [`ember/prefer-ember-test-helpers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/prefer-ember-test-helpers.md)
- [`ember/require-computed-macros`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/require-computed-macros.md)
- [`ember/require-computed-property-dependencies`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/require-computed-property-dependencies.md)
- [`ember/require-return-from-computed`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/require-return-from-computed.md)
- [`ember/require-super-in-lifecycle-hooks`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/require-super-in-lifecycle-hooks.md)
- [`ember/require-tagless-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/require-tagless-components.md)
- [`ember/require-valid-css-selector-in-test-helpers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/require-valid-css-selector-in-test-helpers.md)
- [`ember/use-brace-expansion`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/use-brace-expansion.md)
- [`ember/use-ember-data-rfc-395-imports`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/use-ember-data-rfc-395-imports.md)

Both new rules are in the plugin today as opt-in.

| New rule | Why it belongs in `recommended` |
| ---- | ------------------------------- |
| [`ember/no-builtin-form-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-builtin-form-components.md) | A native `<input>` or `<textarea>` is better than the classic-component `<Input>` or `<Textarea>` wrapper. Named for the next major in [#2060][issue-2060], and implemented in [#2282][pr-2282] |
| [`ember/no-modifier-without-element-usage`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-modifier-without-element-usage.md) | A modifier that never reads its element is an effect that render schedules. `ember/template-no-at-ember-render-modifiers` already rejects that data flow in a template, and it is in `recommended` today. That rule bans one import, and `ember-modifier` reaches the same pattern in three lines |

[pr-2282]: https://github.com/ember-cli/eslint-plugin-ember/pull/2282

## Appendix D: rules removed from `recommended`

The plugin keeps each of these rules. They only come out of `recommended`.

| Rule | Why it is no longer necessary |
| ---- | ----------------------------- |
| `ember/no-implicit-injections` | implicit injections do not exist in ember-source 4+ ([RFC #0680](https://github.com/emberjs/rfcs/blob/master/text/0680-implicit-injection-deprecation.md)). One of the two most expensive rules in the set |
| `ember/no-deprecated-router-transition-methods` | 4.0 removed the deprecated transition methods. The other most expensive rule |
| `ember/avoid-using-needs-in-controllers` | ember-source removed `needs` before 3.x |
| `ember/routes-segments-snake-case` | the motivation for this rule was the implicit route model, which is deprecated ([RFC #0774](https://github.com/emberjs/rfcs/blob/master/text/0774-implicit-record-route-loading.md)). Raised in [#2060][issue-2060] |
| `ember/no-deeply-nested-dependent-keys-with-each` | computed-property era ([#1950][issue-1950]) |
| `ember/no-volatile-computed-properties` | 4.0 removed `.volatile()` |
| `ember/closure-actions` | 3.x-era invocation style |
| `ember/no-function-prototype-extensions` | 4.0 removed prototype extensions |
| `ember/no-old-shims` | ember-cli-shims era |
| `ember/no-string-prototype-extensions` | 4.0 removed prototype extensions |
| `ember/no-get-with-default` | 4.0 removed `getWithDefault` |
| `ember/no-try-invoke` | 4.0 removed `tryInvoke` |
| `ember/no-test-module-for` | ember-qunit removed `moduleFor*` long ago |
| `ember/require-fetch-import` | the blueprint no longer has `ember-fetch` ([RFC #1065](https://github.com/emberjs/rfcs/blob/master/text/1065-remove-ember-fetch.md)). See [#1224][issue-1224] |

[issue-1950]: https://github.com/ember-cli/eslint-plugin-ember/issues/1950
[issue-1224]: https://github.com/ember-cli/eslint-plugin-ember/issues/1224

> [!NOTE]
> A second group of rules lints for `@ember/component`-era patterns: `ember/no-actions-hash`, `ember/no-component-lifecycle-hooks`, `ember/no-attrs-in-components`, `ember/no-observers`, `ember/require-tagless-components`, and `ember/no-classic-components`. Those patterns are still _possible_ today, so these rules stay in `recommended` for this major. They become removal candidates after the deprecation of `@ember/component` (see [RFC #1216](https://github.com/emberjs/rfcs/pull/1216)).
