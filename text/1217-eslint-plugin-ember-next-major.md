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

This RFC defines the `recommended` config for the next major of `eslint-plugin-ember` (v14):
- the template rules that were enabled by default in `ember-template-lint` -- those of them that are applicable to strict mode -- become enabled by default for gjs/gts files. This is the config change that [RFC #1214 "Deprecate ember-template-lint"][rfc-1214] committed us to. The `recommended` config is (and stays) gjs/gts only -- linting `.hbs` files remains opt-in via `template-lint-migration` (the hbs config), which keeps the full `ember-template-lint` parity set
- rules that only exist to catch patterns from `ember-source` 3.x and earlier are removed from `recommended`
- two JS rules that catch patterns which are still possible today are added to `recommended`: `ember/no-builtin-form-components` and `ember/no-modifier-without-element-usage`
- the `recommended-gjs` and `recommended-gts` configs are removed, because `recommended` now carries their rules, scoped per file type

Per the process agreed to in [eslint-plugin-ember#2158][issue-2158], changes to the recommended sets of rules require an RFC -- this is that RFC. Planning for the release itself is tracked in [eslint-plugin-ember#2060][issue-2060].

[rfc-1214]: https://github.com/emberjs/rfcs/pull/1214
[issue-2158]: https://github.com/ember-cli/eslint-plugin-ember/issues/2158
[issue-2060]: https://github.com/ember-cli/eslint-plugin-ember/issues/2060

## Motivation

This major has three motivations:

1. [RFC #1214][rfc-1214] deprecates `ember-template-lint` and unifies all Ember lint rules in `eslint-plugin-ember`. As of `eslint-plugin-ember@13`, every `ember-template-lint` rule has been re-implemented as an `ember/template-*` rule[^no-partial], but none of them are in `recommended` yet -- they were kept opt-in (via the `template-lint-migration` config) so that folks running both tools wouldn't get two errors for every violation. With `ember-template-lint` deprecated, the template rules need to be on by default, or newly generated apps lose lint coverage they've always had -- including the A11y rules, which the Ember project has a core commitment to keeping on by default.

2. The current `recommended` set still spends time linting for patterns that cannot exist in apps on `ember-source` 4+. Some of these rules are also the most expensive rules in the set -- profiling in [eslint-plugin-ember#2060][issue-2060] showed `ember/no-implicit-injections` and `ember/no-deprecated-router-transition-methods` at ~6.2 seconds _each_ (5.9% of total lint time, each) on a large app, checking for things that were removed from ember-source years ago.

3. A major is the only place `recommended` can gain a rule that reports on existing code. Two such rules are ready and are held back only by the semver cost: `ember/no-builtin-form-components` and `ember/no-modifier-without-element-usage`. Both catch patterns that are still reachable in an app on the latest `ember-source`, which is the opposite of the rules being removed in motivation 2.

[^no-partial]: every rule except `no-partial` -- `{{partial}}` was removed from ember-source in 4.0, so there is nothing left to lint against.

## Detailed design

### Add the gjs/gts-applicable template rules to `recommended`

The rules from the existing [`template-lint-migration` config][migration-config] that are applicable to strict-mode templates are added to `recommended`, scoped to `**/*.{gjs,gts}`. The full list is in [Appendix A](#appendix-a-template-rules-added-to-recommended). The baseline is `ember-template-lint`'s `recommended` preset, plus `ember/template-no-template-lint-directives`, which converts leftover `{{! template-lint-disable ... }}` comments to eslint directives via `eslint --fix`.

"Applicable to strict-mode templates" excludes two groups, which stay in the hbs config only ([Appendix B](#appendix-b-loose-mode-only-rules-not-added-to-recommended)):

- `ember/template-no-implicit-this` and `ember/template-no-curly-component-invocation` -- the whole point of these rules is to prepare loose-mode templates for strict mode, and a gjs/gts file is already there. `ember-template-lint`'s own recommended preset disables both for gjs/gts.
- rules that lint constructs which cannot be expressed in strict mode in the first place (`{{action}}`, curly `{{input}}`, `{{#with}}`, the old `{{view}}`/`{{render}}` helpers, ...). These can never fire in a gjs/gts file, so keeping them enabled would only cost lint time -- the same reasoning as the legacy JS rule removals below.

The `recommended` config today is gjs/gts (and js/ts) only, and that does not change -- it never touches `.hbs` files, so adding it to a project can never require also configuring an hbs parser. Newly generated apps have no `.hbs` files, so `recommended` alone gives them full lint parity with what `ember-template-lint` provided.

[migration-config]: https://github.com/ember-cli/eslint-plugin-ember/blob/main/lib/config/template-lint-migration.js

> [!NOTE]
> `ember-template-lint`'s recommended preset also had to disable `builtin-component-arguments`, `no-builtin-form-components`, and `no-unknown-arguments-for-builtin-components` for gjs/gts, because it has no knowledge of imports. The eslint implementations don't have this problem -- they can see the whole module, so (for example) `ember/template-builtin-component-arguments` can check whether `<Input>` is actually the one from `@ember/component`, and not a local component that happens to share the name. Those rules _do_ move to `recommended`. This is one of the motivations of [RFC #1214][rfc-1214].

### Add two JS rules to `recommended`

Both rules already ship in the plugin as opt-in. Neither is a template rule, so both are scoped the way the rest of `recommended` is, not to `**/*.{gjs,gts}`.

| Rule | Why it belongs in `recommended` |
| ---- | ------------------------------- |
| [`ember/no-builtin-form-components`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-builtin-form-components.md) | native `<input>` / `<textarea>` are preferred over the classic-component `<Input>` / `<Textarea>` wrappers. Called out for the next major in [#2060][issue-2060], implemented in [#2282][pr-2282] |
| [`ember/no-modifier-without-element-usage`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-modifier-without-element-usage.md) | a modifier that never references its element is an effect scheduled by render. That is the data flow this RFC's `recommended` set already rejects in templates via `ember/template-no-at-ember-render-modifiers`, and it is just as available through `ember-modifier` directly |

`ember/no-modifier-without-element-usage` covers both modifier styles: the `element` parameter of a `modifier()` callback, and `modify(element)` or `this.element` in a class modifier. Any reference counts, so passing the element to a chart library or a `ResizeObserver` is fine. What it catches is the modifier that exists only to run code at render time, which brings the problems that [`ember/no-at-ember-render-modifiers`](https://github.com/ember-cli/eslint-plugin-ember/blob/master/docs/rules/no-at-ember-render-modifiers.md) documents: extra renders, render loops, and behavior separated from the data it depends on. Those problems do not come from `@ember/render-modifiers` as a package, so a rule that only bans that import leaves the pattern reachable with three lines of `ember-modifier`.

[pr-2282]: https://github.com/ember-cli/eslint-plugin-ember/pull/2282

### Remove the `recommended-gjs` and `recommended-gts` configs

`recommended` now carries the gjs and gts rules itself, in blocks scoped by `files`. Both extra configs became redundant, so v14 removes them.

`recommended-gts` also did one thing that `recommended` did not. It turned off the core rules that TypeScript already reports, and turned on the four that TypeScript makes better (`no-var`, `prefer-const`, `prefer-rest-params`, `prefer-spread`). That set comes from the `eslint-recommended` config of typescript-eslint. `recommended` now applies it to `**/*.gts`.

The result is one config instead of three:

```js
// eslint.config.mjs
import emberRecommended from 'eslint-plugin-ember/configs/recommended';

export default [
  ...emberRecommended,
  // a parser block per file type is still required
];
```

An app that extends `plugin:ember/recommended-gjs` or `plugin:ember/recommended-gts` deletes those two lines. Every rule they set is in `recommended` already.

### Linting `.hbs` files stays opt-in

Existing apps with loose-mode templates opt in with the hbs config -- the [`template-lint-migration` config][migration-config], which keeps the _full_ `ember-template-lint` parity set (including the loose-mode-only rules that don't move to `recommended`) -- plus a parser block routing `.hbs` files to the hbs parser that `eslint-plugin-ember` already ships (via `ember-eslint-parser/hbs`):

```js
// eslint.config.mjs
import { hbsParser, plugin as ember } from 'eslint-plugin-ember/recommended';
import emberTemplateLintMigration from 'eslint-plugin-ember/configs/template-lint-migration';

export default [
  // ... existing config ...
  {
    files: ['app/**/*.hbs'],
    plugins: { ember },
    languageOptions: { parser: hbsParser },
  },
  ...emberTemplateLintMigration,
];
```

> [!IMPORTANT]
> If a config has a `@typescript-eslint/parser` block with a broad `files` glob, that glob must be narrowed to `['**/*.{js,ts,gjs,gts}']` so it doesn't try to parse `.hbs` files -- flat config merges every matching block, and type-aware rules will error on non-JS files.

This config block is documented in the migration guide required by [RFC #1214][rfc-1214], and does not go in the app blueprint -- newly generated apps have no `.hbs` files.

### Remove legacy rules from `recommended`

The following rules are removed from `recommended`. They lint against APIs that no longer exist in `ember-source` 4+, so on any app that can actually install `eslint-plugin-ember@14` they can never fire -- they only cost lint time.

| Rule | Why it's no longer needed |
| ---- | ------------------------- |
| `ember/no-implicit-injections` | implicit injections don't exist in ember-source 4+ ([RFC #0680](https://github.com/emberjs/rfcs/blob/master/text/0680-implicit-injection-deprecation.md)); one of the two most expensive rules in the set |
| `ember/no-deprecated-router-transition-methods` | the deprecated transition methods were removed in 4.0; the other most expensive rule |
| `ember/avoid-using-needs-in-controllers` | `needs` was removed before 3.x |
| `ember/routes-segments-snake-case` | the rule's motivation was the implicit route model, which is deprecated ([RFC #0774](https://github.com/emberjs/rfcs/blob/master/text/0774-implicit-record-route-loading.md)); raised in [#2060][issue-2060] |
| `ember/no-deeply-nested-dependent-keys-with-each` | computed-property era ([#1950][issue-1950]) |
| `ember/no-volatile-computed-properties` | `.volatile()` was removed in 4.0 |
| `ember/closure-actions` | 3.x-era invocation style |
| `ember/no-function-prototype-extensions` | prototype extensions were removed in 4.0 |
| `ember/no-old-shims` | ember-cli-shims era |
| `ember/no-string-prototype-extensions` | prototype extensions were removed in 4.0 |
| `ember/no-get-with-default` | `getWithDefault` was removed in 4.0 |
| `ember/no-try-invoke` | `tryInvoke` was removed in 4.0 |
| `ember/no-test-module-for` | `moduleFor*` was removed from ember-qunit long ago |
| `ember/require-fetch-import` | `ember-fetch` is removed from the blueprint ([RFC #1065](https://github.com/emberjs/rfcs/blob/master/text/1065-remove-ember-fetch.md)); see [#1224][issue-1224] |

[issue-1950]: https://github.com/ember-cli/eslint-plugin-ember/issues/1950
[issue-1224]: https://github.com/ember-cli/eslint-plugin-ember/issues/1224

None of these rules are deleted from the plugin -- they just come out of `recommended`. Apps still working through an older ember-source can re-enable them in their own config:

```js
export default [
  ...emberRecommended,
  {
    rules: {
      'ember/no-get-with-default': 'error',
      // ...whichever of the removed rules still apply
    },
  },
];
```

or simply stay on `eslint-plugin-ember@13` until they're on ember-source 4+.

> [!NOTE]
> There is a second group of rules that lint against `@ember/component`-era patterns (`ember/no-actions-hash`, `ember/no-component-lifecycle-hooks`, `ember/no-attrs-in-components`, `ember/no-observers`, `ember/require-tagless-components`, `ember/no-classic-components`). Those patterns are still _possible_ today, so these rules stay in `recommended` for this major. They become removal candidates once `@ember/component` is deprecated (see [RFC #1216](https://github.com/emberjs/rfcs/pull/1216)).

### What happens to `template-lint-migration`

The config stays, permanently: it is what folks who still have `.hbs` files should use. It keeps the full `ember-template-lint` parity set, so hbs coverage is identical before and after this major, and the [RFC #1214][rfc-1214] migration guide is written around it.

Since it's not really a _migration_ config anymore, v14 should rename it to `hbs` (keeping `template-lint-migration` as an alias for the deprecation window) so that its long-term purpose is reflected in its name.

### Other breaking changes

[#2060][issue-2060] also tracks routine breaking changes for the release (dropping EOL node versions, matching eslint's supported version ranges, config export cleanup). Those don't change what is linted, so they don't need an RFC and aren't specified here -- this RFC is only about the `recommended` rule set. The two config removals above are the exception: they are specified here because folding the typescript-eslint disables into `recommended` changes what is linted in a gts file.

## How we teach this

Most of the teaching work is already required by [RFC #1214][rfc-1214]: the migration guide from `ember-template-lint` covers the `.hbs` parser block, the directive conversion (`template-lint-disable` → `eslint-disable-next-line`), and moving custom rule overrides from `.template-lintrc.js` into `eslint.config.mjs`.

For the v14 upgrade itself:

- Newly generated apps just get the new config from the blueprint. No teaching needed.
- Existing apps upgrading to v14 will see new errors in their gjs/gts files, and in js/ts files from the two added JS rules. The release notes should point at [eslint bulk suppressions](https://eslint.org/blog/2025/04/introducing-bulk-suppressions/) and [Lint to the Future](https://github.com/mansona/lint-to-the-future) for adopting the new rules incrementally instead of fixing everything in one PR. (This replaces the `lint-todo` workflow from `ember-template-lint`, per RFC #1214.)
- Apps with `.hbs` files opt in to hbs linting via the hbs config -- that's a migration-from-`ember-template-lint` task (covered by RFC #1214's migration guide), not a v14-upgrade task.
- Apps that were already using the `template-lint-migration` config see no new template errors at all -- v14 is a no-op for them.

## Drawbacks

- Existing apps get a bunch of new lint errors on upgrade. That's the point of the major (per RFC #1214), and bulk suppressions exist so that nobody has to fix them all in one PR.
- `ember/no-modifier-without-element-usage` has no autofix, and clearing a violation is a refactor rather than a rename: the behavior moves to derived state, a resource, or an event handler on the element that already triggers it. Apps that used modifiers as render hooks will want bulk suppressions for this rule while they work through it.
- Apps still supporting ember-source 3.x silently lose coverage for the removed rules unless they re-enable them. The release notes and the changelog entry for each removed rule cover this -- and staying on v13 is always an option, since nothing about v13 stops working.

## Alternatives

### Keep the template rules opt-in

This contradicts RFC #1214 -- newly generated apps would ship without the default lint coverage (including A11y coverage) that every Ember app has had for years.

### Enable only the A11y subset by default

Preserves the A11y commitment with fewer new errors, but breaks parity with what `ember-template-lint` users have today, and makes "did I migrate correctly?" harder to answer. Parity is easier to explain: same rules, one tool.

### Ship a `legacy` config with the removed rules

[#1950][issue-1950] proposed this. It's another config to name, document, and maintain, for a set of apps that shrinks every year -- and those apps already have two options: stay on v13, or re-enable the handful of rules they still want in their own config. Not worth maintaining.

### Keep `ember/no-modifier-without-element-usage` opt-in and rely on `ember/no-at-ember-render-modifiers`

`ember/no-at-ember-render-modifiers` is already in `recommended`, so the argument is that the pattern is covered. It is not: that rule bans an import, and the same effect-on-render pattern is available by writing the modifier by hand. An app that removes `@ember/render-modifiers` by reimplementing `{{did-insert}}` locally passes today's `recommended` set.

### Merge the template rules into `recommended` unscoped (including `.hbs`)

Then `recommended` would blow up for anyone with `.hbs` files who hasn't configured the hbs parser. Keeping `recommended` gjs/gts-only means it works without any parser setup, and hbs linting stays opt-in.

## Unresolved questions

- The Appendix A / Appendix B split is a best guess -- during implementation, each rule should be checked against what strict mode can actually express (some loose-mode constructs, like `{{unbound}}` or `{{mut}}`, are still keywords in strict mode, so their rules stay in Appendix A). Rules that end up in the wrong list move without re-RFCing.
- Should `ember/template-no-bare-strings` (off by default in `ember-template-lint`, off by default here) get any special mention in the migration guide for i18n-heavy apps? (Leaning yes, but it's a docs question, not a config question.)

## Appendix A: template rules added to `recommended`

87 rules, at `error`, scoped to `**/*.{gjs,gts}`. This is the `template-lint-migration` rule set (`ember-template-lint`'s recommended preset -- minus `no-partial`, which has no equivalent because `{{partial}}` no longer exists -- plus `template-no-template-lint-directives`) with the loose-mode-only rules from Appendix B taken out.

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

These 9 rules stay in the hbs config (`template-lint-migration`), keeping full `ember-template-lint` parity for `.hbs` files.

| Rule | Why it doesn't apply to gjs/gts |
| ---- | ------------------------------- |
| [`ember/template-no-implicit-this`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-implicit-this.md) | strict mode has no implicit `this` -- an unresolved reference is a build error |
| [`ember/template-no-curly-component-invocation`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-curly-component-invocation.md) | the component-or-property ambiguity doesn't exist in strict mode |
| [`ember/template-no-action`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-action.md) | `{{action}}` has no strict-mode import |
| [`ember/template-no-route-action`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-route-action.md) | `{{route-action}}` has no strict-mode import |
| [`ember/template-no-input-block`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-input-block.md) | curly `{{input}}` cannot resolve in strict mode |
| [`ember/template-no-input-tagname`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-input-tagname.md) | curly `{{input}}` cannot resolve in strict mode |
| [`ember/template-no-with`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-no-with.md) | `{{#with}}` was removed in ember-source 4.0 and cannot be expressed in strict mode |
| [`ember/template-deprecated-render-helper`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-deprecated-render-helper.md) | `{{render}}` is long gone and cannot be expressed in strict mode |
| [`ember/template-deprecated-inline-view-helper`](https://github.com/ember-cli/eslint-plugin-ember/blob/6f89075805fdf4487d3f3631fbed58f5e1b8bdab/docs/rules/template-deprecated-inline-view-helper.md) | `{{view}}` is long gone and cannot be expressed in strict mode |
