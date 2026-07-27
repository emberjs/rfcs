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

Per the process agreed to in [eslint-plugin-ember#2158][issue-2158], changes to the recommended sets of rules require an RFC -- this is that RFC. Planning for the release itself is tracked in [eslint-plugin-ember#2060][issue-2060].

[rfc-1214]: https://github.com/emberjs/rfcs/pull/1214
[issue-2158]: https://github.com/ember-cli/eslint-plugin-ember/issues/2158
[issue-2060]: https://github.com/ember-cli/eslint-plugin-ember/issues/2060

## Motivation

This major has two motivations:

1. [RFC #1214][rfc-1214] deprecates `ember-template-lint` and unifies all Ember lint rules in `eslint-plugin-ember`. As of `eslint-plugin-ember@13`, every `ember-template-lint` rule has been re-implemented as an `ember/template-*` rule[^no-partial], but none of them are in `recommended` yet -- they were kept opt-in (via the `template-lint-migration` config) so that folks running both tools wouldn't get two errors for every violation. With `ember-template-lint` deprecated, the template rules need to be on by default, or newly generated apps lose lint coverage they've always had -- including the A11y rules, which the Ember project has a core commitment to keeping on by default.

2. The current `recommended` set still spends time linting for patterns that cannot exist in apps on `ember-source` 4+. Some of these rules are also the most expensive rules in the set -- profiling in [eslint-plugin-ember#2060][issue-2060] showed `ember/no-implicit-injections` and `ember/no-deprecated-router-transition-methods` at ~6.2 seconds _each_ (5.9% of total lint time, each) on a large app, checking for things that were removed from ember-source years ago.

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

One more addition: `ember/no-builtin-form-components` (called out for the next major in [eslint-plugin-ember#2060][issue-2060], implemented in [#2282][pr-2282]) -- native `<input>` / `<textarea>` are preferred over the classic-component `<Input>` / `<Textarea>` wrappers.

[pr-2282]: https://github.com/ember-cli/eslint-plugin-ember/pull/2282

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

[#2060][issue-2060] also tracks routine breaking changes for the release (dropping EOL node versions, matching eslint's supported version ranges, config export cleanup). Those don't change what is linted, so they don't need an RFC and aren't specified here -- this RFC is only about the `recommended` rule set.

## How we teach this

Most of the teaching work is already required by [RFC #1214][rfc-1214]: the migration guide from `ember-template-lint` covers the `.hbs` parser block, the directive conversion (`template-lint-disable` → `eslint-disable-next-line`), and moving custom rule overrides from `.template-lintrc.js` into `eslint.config.mjs`.

For the v14 upgrade itself:

- Newly generated apps just get the new config from the blueprint. No teaching needed.
- Existing apps upgrading to v14 will see new errors in their gjs/gts files. The release notes should point at [eslint bulk suppressions](https://eslint.org/blog/2025/04/introducing-bulk-suppressions/) and [Lint to the Future](https://github.com/mansona/lint-to-the-future) for adopting the new rules incrementally instead of fixing everything in one PR. (This replaces the `lint-todo` workflow from `ember-template-lint`, per RFC #1214.)
- Apps with `.hbs` files opt in to hbs linting via the hbs config -- that's a migration-from-`ember-template-lint` task (covered by RFC #1214's migration guide), not a v14-upgrade task.
- Apps that were already using the `template-lint-migration` config see no new template errors at all -- v14 is a no-op for them.

## Drawbacks

- Existing apps get a bunch of new lint errors on upgrade. That's the point of the major (per RFC #1214), and bulk suppressions exist so that nobody has to fix them all in one PR.
- Apps still supporting ember-source 3.x silently lose coverage for the removed rules unless they re-enable them. The release notes and the changelog entry for each removed rule cover this -- and staying on v13 is always an option, since nothing about v13 stops working.

## Alternatives

### Keep the template rules opt-in

This contradicts RFC #1214 -- newly generated apps would ship without the default lint coverage (including A11y coverage) that every Ember app has had for years.

### Enable only the A11y subset by default

Preserves the A11y commitment with fewer new errors, but breaks parity with what `ember-template-lint` users have today, and makes "did I migrate correctly?" harder to answer. Parity is easier to explain: same rules, one tool.

### Ship a `legacy` config with the removed rules

[#1950][issue-1950] proposed this. It's another config to name, document, and maintain, for a set of apps that shrinks every year -- and those apps already have two options: stay on v13, or re-enable the handful of rules they still want in their own config. Not worth maintaining.

### Merge the template rules into `recommended` unscoped (including `.hbs`)

Then `recommended` would blow up for anyone with `.hbs` files who hasn't configured the hbs parser. Keeping `recommended` gjs/gts-only means it works without any parser setup, and hbs linting stays opt-in.

## Unresolved questions

- The Appendix A / Appendix B split is a best guess -- during implementation, each rule should be checked against what strict mode can actually express (some loose-mode constructs, like `{{unbound}}` or `{{mut}}`, are still keywords in strict mode, so their rules stay in Appendix A). Rules that end up in the wrong list move without re-RFCing.
- Should `ember/template-no-bare-strings` (off by default in `ember-template-lint`, off by default here) get any special mention in the migration guide for i18n-heavy apps? (Leaning yes, but it's a docs question, not a config question.)

## Appendix A: template rules added to `recommended`

87 rules, at `error`, scoped to `**/*.{gjs,gts}`. This is the `template-lint-migration` rule set (`ember-template-lint`'s recommended preset -- minus `no-partial`, which has no equivalent because `{{partial}}` no longer exists -- plus `template-no-template-lint-directives`) with the loose-mode-only rules from Appendix B taken out.

- `ember/template-builtin-component-arguments`
- `ember/template-link-href-attributes`
- `ember/template-link-rel-noopener`
- `ember/template-no-abstract-roles`
- `ember/template-no-accesskey-attribute`
- `ember/template-no-action-on-submit-button`
- `ember/template-no-args-paths`
- `ember/template-no-arguments-for-html-elements`
- `ember/template-no-aria-hidden-body`
- `ember/template-no-aria-unsupported-elements`
- `ember/template-no-array-prototype-extensions`
- `ember/template-no-at-ember-render-modifiers`
- `ember/template-no-attrs-in-components`
- `ember/template-no-autofocus-attribute`
- `ember/template-no-block-params-for-html-elements`
- `ember/template-no-builtin-form-components`
- `ember/template-no-capital-arguments`
- `ember/template-no-class-bindings`
- `ember/template-no-debugger`
- `ember/template-no-duplicate-attributes`
- `ember/template-no-duplicate-id`
- `ember/template-no-duplicate-landmark-elements`
- `ember/template-no-empty-headings`
- `ember/template-no-extra-mut-helper-argument`
- `ember/template-no-forbidden-elements`
- `ember/template-no-heading-inside-button`
- `ember/template-no-html-comments`
- `ember/template-no-index-component-invocation`
- `ember/template-no-inline-styles`
- `ember/template-no-invalid-aria-attributes`
- `ember/template-no-invalid-interactive`
- `ember/template-no-invalid-link-text`
- `ember/template-no-invalid-link-title`
- `ember/template-no-invalid-meta`
- `ember/template-no-invalid-role`
- `ember/template-no-link-to-positional-params`
- `ember/template-no-link-to-tagname`
- `ember/template-no-log`
- `ember/template-no-negated-condition`
- `ember/template-no-nested-interactive`
- `ember/template-no-nested-landmark`
- `ember/template-no-nested-splattributes`
- `ember/template-no-obscure-array-access`
- `ember/template-no-obsolete-elements`
- `ember/template-no-outlet-outside-routes`
- `ember/template-no-passed-in-event-handlers`
- `ember/template-no-pointer-down-event-binding`
- `ember/template-no-positional-data-test-selectors`
- `ember/template-no-positive-tabindex`
- `ember/template-no-potential-path-strings`
- `ember/template-no-quoteless-attributes`
- `ember/template-no-redundant-fn`
- `ember/template-no-redundant-role`
- `ember/template-no-scope-outside-table-headings`
- `ember/template-no-shadowed-elements`
- `ember/template-no-template-lint-directives`
- `ember/template-no-triple-curlies`
- `ember/template-no-unbalanced-curlies`
- `ember/template-no-unbound`
- `ember/template-no-unknown-arguments-for-builtin-components`
- `ember/template-no-unnecessary-component-helper`
- `ember/template-no-unnecessary-curly-parens`
- `ember/template-no-unnecessary-curly-strings`
- `ember/template-no-unsupported-role-attributes`
- `ember/template-no-unused-block-params`
- `ember/template-no-valueless-arguments`
- `ember/template-no-whitespace-for-layout`
- `ember/template-no-whitespace-within-word`
- `ember/template-no-yield-only`
- `ember/template-no-yield-to-default`
- `ember/template-require-aria-activedescendant-tabindex`
- `ember/template-require-button-type`
- `ember/template-require-context-role`
- `ember/template-require-has-block-helper`
- `ember/template-require-iframe-title`
- `ember/template-require-input-label`
- `ember/template-require-lang-attribute`
- `ember/template-require-mandatory-role-attributes`
- `ember/template-require-media-caption`
- `ember/template-require-presentational-children`
- `ember/template-require-valid-alt-text`
- `ember/template-require-valid-named-block-naming-format`
- `ember/template-simple-modifiers`
- `ember/template-simple-unless`
- `ember/template-splat-attributes-only`
- `ember/template-style-concatenation`
- `ember/template-table-groups`

## Appendix B: loose-mode-only rules, not added to `recommended`

These 9 rules stay in the hbs config (`template-lint-migration`), keeping full `ember-template-lint` parity for `.hbs` files.

| Rule | Why it doesn't apply to gjs/gts |
| ---- | ------------------------------- |
| `ember/template-no-implicit-this` | strict mode has no implicit `this` -- an unresolved reference is a build error |
| `ember/template-no-curly-component-invocation` | the component-or-property ambiguity doesn't exist in strict mode |
| `ember/template-no-action` | `{{action}}` has no strict-mode import |
| `ember/template-no-route-action` | `{{route-action}}` has no strict-mode import |
| `ember/template-no-input-block` | curly `{{input}}` cannot resolve in strict mode |
| `ember/template-no-input-tagname` | curly `{{input}}` cannot resolve in strict mode |
| `ember/template-no-with` | `{{#with}}` was removed in ember-source 4.0 and cannot be expressed in strict mode |
| `ember/template-deprecated-render-helper` | `{{render}}` is long gone and cannot be expressed in strict mode |
| `ember/template-deprecated-inline-view-helper` | `{{view}}` is long gone and cannot be expressed in strict mode |
