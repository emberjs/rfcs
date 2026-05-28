---
stage: accepted
start-date: # In format YYYY-MM-DDT00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
  - learning
  - steering
  - typescript
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
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

# `isolatedDeclarations` for libraries

## Summary

Turn on `isolatedDeclarations` in `@ember/library-tsconfig`, the base config that v2 addons extend from. Library authors will need to annotate the types of their exports. In return, type-checking gets faster, the public API of an addon becomes an explicit contract instead of whatever `tsc` happened to infer, and non-`tsc` tools like `tsdown` and `rolldown` can generate `.d.ts` files directly from source.

## Motivation

`@ember/library-tsconfig` is the config every v2 addon inherits. What goes in it is the ecosystem default. The defaults in it should be the defaults that make sense for an Ember _library_, not an Ember _app_.

Apps are the end-consumers. Whatever `(ember-)tsc` infers is what gets used at the call site. Inference is fine because there's nobody downstream to surprise.

Libraries are not end-consumers. Their exports are a contract with other packages: if the type of an export is inferred from internal code, then changing an internal helper can silently change the public API. Nobody reviewing the diff sees it -- the source for the export didn't change -- and consumers find out unexpectedly in a renovate PR or other change that is supposed to be "easy".

`isolatedDeclarations` fixes this by requiring that the `.d.ts` for any exported declaration be computable from just that file. The author writes the public type, preventing any type leakage.

Three benefits:

1. Public API contracts are explicit. Reviewers see the public surface in the same diff as the implementation.

2. Faster `.d.ts` generation by tools that aren't `(ember-)tsc`. [`tsdown` already uses `oxc-transform` for this](https://tsdown.dev/options/dts#with-isolateddeclarations) and recommends `isolatedDeclarations` for it so that the developer has a pre-build-time way to verify that source will be compatible with in-AST declaration derivation.

3. Type-checking is faster. Less inference means less cross-file work for the type checker. The effect shows up in the editor on large addons (and apps).

## Detailed design

### What changes in the config

`@ember/library-tsconfig/tsconfig.json` gets [`isolatedDeclarations`](https://www.typescriptlang.org/tsconfig/#isolatedDeclarations):

```jsonc
{
  "compilerOptions": {
    "isolatedDeclarations": true
  }
}
```

This is the change that landed in [ember-cli/tsconfigs#19](https://github.com/ember-cli/tsconfigs/pull/19) and was then reverted in [#22](https://github.com/ember-cli/tsconfigs/pull/22) pending this RFC.

### What authors need to write differently

> [!NOTE]
> This is not specific to ember, but instead: general TypeScript authoring.

Return types on exported functions:

```ts
// before -- allowed today, errors under isolatedDeclarations
export function joined(val: string[]) {
  return val.join(' ');
}

// after
export function joined(val: string[]): string {
  return val.join(' ');
}
```

Annotations on exported `const`s whose type comes from a non-trivial expression:

```ts
// before
export const config = makeConfig({ /* ... */ });

// after
export const config: Config = makeConfig({ /* ... */ });
```

Literal initializers like `export const x = 1` are still fine. The rule only kicks in when the type can't be read straight off the syntax.

Some patterns need restructuring. The symbol-keyed brand below can't be expressed under `isolatedDeclarations` because the symbol's type isn't a literal:

```ts
const IS_OPTGROUP = Symbol('isOptGroup');

export interface OptGroup {
  readonly [IS_OPTGROUP]: true; // not expressible
}
```

The fix is to either export the symbol (so the brand can be reconstructed in the `.d.ts`) or use a different branding technique (nominal type, declared `unique symbol`, etc.). This is the most disruptive case for existing addons that use this pattern, and there aren't that many of them.

> [!NOTE]
> This is a TypeScript limitation, but the benefits are worth dealing with it 

### Migration

This is a breaking change to `@ember/library-tsconfig`. Three options for an existing addon:

- Add the missing boundary-types. TypeScript's error messages for `isolatedDeclarations` are good and point at exactly what's missing.
- Stay on the previous major of `@ember/library-tsconfig`.
- Opt out. Set `isolatedDeclarations: false` in the addon's own `tsconfig.json`. 

> [!NOTE]
> While ember defaults can't force anyone to do anything with TypeScript, it is important to understand the consequences of various styles of authored declarations.

### Ecosystem implications

- Existing v2 addons will see new errors on upgrade if they have missed any boundary types.
- Blueprints. v2 addon blueprint needs the project-references layout described above so new addons get editor-time enforcement out of the box.
- IDE. No editor config changes needed. Errors show up in any editor that uses `tsserver`.
- Lint. No new ESLint rule. `tsc` enforces this directly.
- `ember-tsc` and future tooling. This RFC is what unblocks `rolldown`- and `oxc-transform`-based `.d.ts` generation in Ember tooling. Without `isolatedDeclarations`, those tools fall back to `tsc`, which is the thing they're trying to replace.
- SSR, Inspector, Engines. No impact.

## How we teach this

The library guides, which don't exist yet, should talk about the various tsconfig options, and their impact on the build for the library as well as the impact on type checking and subsequent builds for consuming projects.

## Drawbacks

- Existing addons will see errors on upgrade -- however, this is not uncommon for "breaking change releases" of any package - semver is a good tool. 
- A few patterns (symbol brands, types derived from expressions) need restructuring. Workarounds exist.
- Some stylistically prefer inference when they work (still good for app dev)

## Alternatives

Leave `isolatedDeclarations` out of `@ember/library-tsconfig` -- add it directly to the v2 addon blueprint's tsconfig(s).

Only set it in `tsconfig.publish.json`. This is what some addons do today. Errors show up at publish, not in the editor (and is is the simplest to implement). 

## Unresolved questions

- Errors need to show up in the editor, not at publish time.
    With the current v2 addon blueprint setup, there are multiple tsconfig.jsons (one for dev and one for publish). Ideally, isolatedDeclarations' errors would show up in-editor, but the specifics of how will need to be explored.

## Appendix and related reading

- [Isolated Declarations and Zod](https://v5.chriskrycho.com/notes/isolated-declarations-and-zod/)
- [Speeding up the JavaScript ecosystem - Isolated Declarations](https://marvinh.dev/blog/speeding-up-javascript-ecosystem-part-10/)
