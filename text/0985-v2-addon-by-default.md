---
stage: accepted
start-date: 2023-11-02T16:05:11.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams: # delete teams that aren't relevant
  - cli
  - data
  - framework
prs:
  accepted: https://github.com/emberjs/rfcs/pull/985
project-link:
suite:
---

# Change the default addon blueprint to `@ember/addon-blueprint`

## Summary

This RFC proposes making [`@ember/addon-blueprint`](https://github.com/ember-cli/ember-addon-blueprint) the default blueprint for new Ember addons, replacing the current v1 and v2 blueprints. The existing blueprints present significant technical challenges that impact developer productivity and ecosystem compatibility. The new blueprint addresses these issues through modern tooling, streamlined architecture, and comprehensive TypeScript integration based on extensive community feedback and production usage patterns.

## Motivation

The current default v1 addon blueprint generates addons that get rebuilt by every consuming app, which is slow and couples addon builds to app builds. There is no built-in path to modern tooling like TypeScript, Glint, or Vite. The app blueprint already defaults to a modern Vite-based setup, so the addon blueprint is out of parity -- new addon authors get a significantly worse starting experience than new app authors.

`@ember/addon-blueprint` already exists and has been widely adopted by the community. Making it the default gives new addon authors a working setup with single-package structure, Glint for template type safety, native classes and strict mode throughout, and sensible tooling defaults out of the box.

Goals:
- The new blueprint should :
  - emit browser-code by default (engines / node doesn't matter)
  - use standard ecosystem tooling with broad support
  - be familiar / easy to test / make demo apps with
  - document and explain how the npm packaging process works, and what concerns someone publishing a package needs to be aware of
  - have escape hatches / configuration for folks who don't need everything that a publisher would need
  - compose within monorepos

> [!NOTE]
> the point about publishing concerns could be the most contentious, as some folks feel we have too many config files. And while there are more than a v1 addon, they are _needed_ when your develop and publish environments/targets are different. More on that later.

## Detailed design

### Definitions

**V2 Addon**: An addon with `ember-addon.version: 2` in package.json, as defined by [RFC 507](https://rfcs.emberjs.com/id/0507-embroider-v2-package-format/).

**Single-package addon**: An addon with its test suite in the same package, rather than a separate test app in a monorepo.

**Blueprint**: A code generation template used by ember-cli to scaffold projects.

### Migration

The majority of this RFC is written from the perspective of someone that is running ember new for the first time on a brand new addon, but we will need to make sure to write both upgrade guides and appropriate codemods for anyone that is wanting to upgrade their addons from the old blueprints to this new default. We also need to put some consideration into the experience of people using `ember-cli-update` to upgrade when we make this the new default. 

#### `ember-cli-update` Support

The blueprint includes `config/ember-cli-update.json` so that `ember-cli-update` continues to work. This file tracks the blueprint package name and version, allowing `ember-cli-update` to detect available updates and apply them. The entry should reference `@ember/addon-blueprint` and the version used to generate the addon, following the same pattern used by the app blueprint.

Note that there will be a version boundary across which `ember-cli-update` cannot automatically migrate. Addons generated with the old v1 blueprint cannot be automatically updated to the new `@ember/addon-blueprint` via `ember-cli-update` -- the project structures are too different. Similar to how apps needed to reach a specific ember-cli version before `ember-cli-update` could bridge to the Vite-based app blueprint, addon authors will need to do a one-time manual migration (or use a codemod, once available) to get onto the new blueprint. Once on the new blueprint, `ember-cli-update` will work normally for subsequent updates.

#### Codemod

A codemod for migrating existing v1 addons to the new blueprint structure is out of scope for this RFC, but would be a valuable follow-up effort. [Mainmatter](https://mainmatter.com/) has expressed interest in developing such a codemod. In the meantime, addon authors can generate a fresh project with the new blueprint and manually move their source code into it. The [embroider-build/embroider](https://github.com/embroider-build/embroider) repo also has documentation on how to work with and migrate to v2 addons.


### In-repo addons

_Proper_ in-repo addons, are not going to be supported with this blueprint.

`ember-cli` will throw an error _unless_ the user explicitly opts in to tho old blueprint. 

### Generators

The new blueprint does not include `ember-cli` as a dependency. This means commands like `ember generate component foo` will not work out of the box in a v2 addon:

```
# pnpm dlx ember-cli g component foo

You have to be inside an ember-cli project to use the generate command.
```

_v2 addons are not ember-cli projects_.

> [!IMPORTANT]
> This is expected behavior, however before we mark this "v2 addon by default" project as "ready for release", we'll want _some way_ to help folks generate files. 



## How we teach this

### Documentation Updates

- Update the Ember Guides to reference the new blueprint
- The blueprint README covers customization, publishing, and multi-version support
- Provide migration guides for v1 and v2 addon authors
- The blueprint should generate parallel `.md` files (or inline comments) alongside config files to explain the purpose and rationale of each configuration. This helps addon authors understand *why* a config exists, not just *what* it contains, and reduces confusion when configs change across blueprint versions
- Review the existing Ember Guides to identify workflows that won't work with the new blueprint and document alternatives

### Key Concepts for Addon Authors

#### `exports` and `imports`

> [!NOTE]
> This may look more complicated than v1 addons, but this is what the wider ecosystem is doing, and a lot of tooling knows to look at these package.json configs, and we'll benefit from that tooling.

`exports` defines your addon's public API:

```json
{
  "exports": {
    ".": {
      "types": "./declarations/index.d.ts",
      "default": "./dist/index.js"
    },
    "./*.css": "./dist/*.css",
    "./*": {
      "types": "./declarations/*.d.ts",
      "default": "./dist/*.js"
    }
  }
}
```

`imports` with `#src/*` lets tests and the demo app import from source without rebuilding. Can't be used in `src/` (Rollup won't transform these). Files in `src/` must use relative imports.

#### Importing Addon Code in Tests: `#src/*` vs. Consumer-Style

When writing tests, you have two ways to import from your addon:

**`#src/*` imports** -- import directly from source files:
```javascript
import { myHelper } from '#src/helpers/my-helper';
```
- Works immediately, no build step needed
- Fast feedback loop during development
- Tests the source code directly

**Consumer-style imports** -- import as a consumer would:
```javascript
import { myHelper } from 'my-addon/helpers/my-helper';
```
- Tests the published API surface
- Requires `dist/` to exist (needs a build first)
- Catches issues with `exports` mapping or build transforms

#### Self-Imports

Self-imports (e.g. `import { x } from 'my-addon/foo'`) don't work during development in `src/` files because they resolve through `exports` to `dist/`, which doesn't exist until you build. Use relative imports in `src/`:

```javascript
// In src/ files:
import { myHelper } from './helpers/my-helper'; // yes
import { myHelper } from 'my-addon/helpers/my-helper'; // no
```

> [!NOTE]
> self-imports _can_ work with custom export conditions coordinated between package.json, rollup, and vite configs.

#### Dev vs. Publish Configs

| Purpose | Dev | Publish |
|---------|-----|---------|
| Babel | `babel.config.cjs` (macros, compat) | `babel.publish.config.cjs` (minimal) |
| TypeScript | `tsconfig.json` (all files, Vite types) | `tsconfig.publish.json` (src only) |
| Build | `vite.config.mjs` (HMR, tests) | `rollup.config.mjs` (tree-shaking) |

Macros are evaluated in dev for testing but left unevaluated in published output -- the consuming app handles them. The publish tsconfig omits Vite/Embroider types so `lint:types` catches accidental usage.

#### Monorepo Setup


The single-package default works for most addons. If you need a monorepo (complex integration testing, multiple related packages, full documentation app):

> [!NOTE]
> This is an example monorepo setup, and may differ from what your monorepo. Some addons may want _multiple_ test-apps for various configurations / build setups.

1. Generate addon: `npx ember-cli@latest addon my-addon --blueprint @ember/addon-blueprint --skip-git`
2. Generate test app: `npx ember-cli@latest app test-app --blueprint @ember/app-blueprint`
3. Set up workspace tooling (pnpm workspaces)
4. Install addon in test app

#### Unpublished Addons in a Monorepo

Sometimes you have a v2 addon in a monorepo that's only consumed by other packages in the workspace -- it's never published to npm. In this case you can skip the build step entirely and point `exports` at your source files:

```json
{
  "name": "my-internal-addon",
  "ember-addon": {
    "version": 2,
    "type": "addon",
    "main": "addon-main.cjs"
  },
  "exports": {
    ".": {
      "types": "./src/index.ts",
      "default": "./src/index.ts"
    },
    "./*": {
      "types": "./src/*",
      "default": "./src/*"
    }
  }
}
```

Key differences from a published addon:
- `exports` points to `src/` instead of `dist/` and `declarations/`
- No `files` array needed (not publishing to npm)
- No rollup build, no `prepack` script, no `declarations/` directory
- No `babel.publish.config.cjs` or `tsconfig.publish.json` needed
- You still need `addon-main.cjs` if any consuming app in the workspace uses the classic ember-cli build

The consuming app's build tooling (Vite/Embroider) handles the transpilation. This is much simpler to maintain for workspace-internal code.

#### In-Repo Addons

Classic in-repo addons (the `lib/` directory pattern) are v1 constructs. To create the v2 equivalent, use your package manager's workspace features to establish them as real package dependencies:

1. Create a directory for the addon (e.g. `packages/my-internal-addon/` or keep `lib/my-internal-addon/`)
2. Give it a `package.json` with `ember-addon.version: 2`
3. Add it to your workspace configuration (e.g. pnpm `workspace.yaml` or `package.json` `"workspaces"`)
4. Install it as a dependency of the consuming app

These in-repo addons will typically be "unbuilt" -- they point `exports` at source files as described in the Unpublished Addons section above. This avoids the need for a separate build step while still giving you proper package boundaries and clean imports. The consuming app's Vite/Embroider build handles all transpilation.

#### Publishing

1. Write code in `src/`, tests with `#src/*` imports
2. `npm run build` runs Rollup with publish configs, producing `dist/` and `declarations/`
3. `npm publish` ships only `files` from package.json
4. Consumers import via `exports`, not internal paths

### Resources

- [@ember/addon-blueprint README](https://github.com/ember-cli/ember-addon-blueprint#readme)
- [Addon Author Guide](https://github.com/embroider-build/embroider/blob/main/docs/addon-author-guide.md)
- [Porting Addons to V2](https://github.com/embroider-build/embroider/blob/main/docs/porting-addons-to-v2.md)
- [Node.js Package Exports](https://nodejs.org/api/packages.html#exports)
- [Glint Documentation](https://typed-ember.gitbook.io/glint/)

## Drawbacks

- Some advanced use cases (monorepos, custom builds) need additional configuration.
- Addon authors unfamiliar with TypeScript/Glint face a learning curve, but JavaScript is fully supported.
- The blueprint is opinionated, but covers the vast majority of use cases.
- hbs is not supported in addons (it's not possible to use Glint 2 with hbs)

## Alternatives

- Do nothing -- this should have shipped years ago. The community has already broadly adopted v2 addons as the de facto default; the official defaults are lagging behind actual community practice.
- Default to monorepo (too complex for most users (and maintainers of the bluleprints, as it turns out))
- Provide multiple blueprints (maintenance burden, confusion)
  - this is slightly addressed by documenting how to compose blueprints for differentt workflows, like having multiple test apps, for example.
- we can make hbs work without ts support
  - or we can document how to hand-roll types for these


## Appendix, related information, goals, direction

### Influence on Future App Blueprint

The test setup here could be a proof-of-concept for future compat-less Ember apps:

- A minimal Ember app running on Vite with no webpack or `ember-cli-build.js`
- Bootstrap with just `EmberApp` and `EmberRouter` -- no complex build pipeline
- ES modules and `import.meta.glob` for module discovery instead of AMD/requirejs
- Direct framework API usage instead of ember-cli abstractions

This validates that Ember apps can run well on modern build tools, pointing toward simpler app blueprints in the future.

### Non-Built Addons

The blueprint should support a flag (e.g. `--no-build`) that generates a simpler addon intended for local/workspace use rather than publishing to npm. A non-built addon skips the publish-time build entirely -- `exports` points at source files, and the consuming app's build tooling (Vite/Embroider) handles transpilation.

This variant drops a significant chunk of devDependencies and config files that only exist to support the publish build.

The exact flag name and implementation are details of the addon-blueprint itself, not this RFC. The important thing is that the blueprint provides a first-class path for this use case, since local/workspace addons that don't need their own build are common and should not require manually stripping down a full publishable addon scaffold.

