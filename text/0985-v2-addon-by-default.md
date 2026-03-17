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

Make [`@ember/addon-blueprint`](https://github.com/ember-cli/ember-addon-blueprint) the default blueprint for new Ember addons, replacing the current v1 and v2 blueprints.

## Motivation

The current default blueprints have real problems:

**V1 addons** get rebuilt by every consuming app, which is slow and couples addon builds to app builds.

**The original V2 blueprint** fixed the build problem but required a monorepo and manual configuration. It also lacked TypeScript/Glint integration and modern Ember patterns.

`@ember/addon-blueprint` fixes these by defaulting to a single-package structure, integrating Glint for template type safety, using native classes and strict mode throughout, and providing sensible tooling defaults out of the box.

## Detailed design

### Definitions

**V2 Addon**: An addon with `ember-addon.version: 2` in package.json, as defined by RFC 507.

**Single-package addon**: An addon with its test suite in the same package, rather than a separate test app in a monorepo.

**Blueprint**: A code generation template used by ember-cli to scaffold projects.

### Blueprint Structure

```
my-addon/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # CI pipeline
│       └── push-dist.yml             # Publish to dist branch
├── src/                              # Source code (published)
│   ├── index.js                      # Main entry point
│   └── template-registry.ts          # Glint type registry
├── tests/                            # Test files
│   ├── index.html                    # Test runner page
│   └── test-helper.ts                # Test setup
├── demo-app/                         # Demo application
│   ├── app.gts                       # Demo app entry point
│   ├── styles.css
│   └── templates/
├── unpublished-development-types/    # Dev-only types
│   └── index.d.ts
├── config/
│   └── ember-cli-update.json         # Blueprint version tracking
├── dist/                             # Built output (gitignored, published)
├── declarations/                     # TS declarations (gitignored, published)
├── package.json
├── index.html                        # Demo app entry page
├── rollup.config.mjs                 # Production build
├── vite.config.mjs                   # Dev build + tests
├── tsconfig.json                     # Dev TypeScript config
├── tsconfig.publish.json             # Publish TypeScript config
├── babel.config.cjs                  # Dev Babel config
├── babel.publish.config.cjs          # Publish Babel config
├── eslint.config.mjs                 # ESLint flat config
├── .prettierrc.mjs                   # Prettier config
├── .prettierignore
├── .template-lintrc.mjs              # Template linting
├── testem.cjs                        # Test runner config
├── .try.mjs                          # Ember version scenarios
├── .editorconfig
├── .env.development
├── .gitignore
├── README.md
├── CONTRIBUTING.md
├── LICENSE.md
└── addon-main.cjs                    # V1 compat shim
```

**Why single-package by default**: Most addons don't need a monorepo. Single-package is simpler to maintain and publish. Advanced users can still set up monorepos.

**Why dual build systems**: Vite gives fast dev rebuilds and HMR. Rollup gives optimized, tree-shaken production output. Tests run entirely through Vite, no webpack or `ember-auto-import` needed.

**Why Glint**: Template type safety via Volar-based TS server plugins. Works for both TypeScript and JavaScript projects.

### Package Configuration

> [!NOTE]
> This is an overview. Exact contents can change as needs evolve. We aim to cover the 80% use case without restricting anyone who needs more.

#### package.json

```json
{
  "name": "<%= name %>",
  "version": "0.0.0",
  "description": "The default blueprint for Embroider v2 addons.",
  "keywords": ["ember-addon"],
  "repository": "",
  "license": "MIT",
  "author": "",
  "files": [
    "addon-main.cjs",
    "declarations",
    "dist",
    "src"
  ],
  "ember": {
    "edition": "octane"
  },
  "ember-addon": {
    "version": 2,
    "type": "addon",
    "main": "addon-main.cjs"
  },
  "imports": {
    "#src/*": "./src/*"
  },
  "exports": {
    ".": {
      "types": "./declarations/index.d.ts",
      "default": "./dist/index.js"
    },
    "./addon-main.js": "./addon-main.cjs",
    "./*.css": "./dist/*.css",
    "./*": {
      "types": "./declarations/*.d.ts",
      "default": "./dist/*.js"
    }
  }
}
```

`exports` maps consumer-facing imports to the right files (declarations for types, dist for runtime). `imports` with `#src/*` gives tests and the demo app a clean way to import from source without rebuilding -- but can't be used in `src/` itself because Rollup won't transform those imports.

The `files` array includes `src` alongside `dist` and `declarations` so consumers get source-level go-to-definition in their editors.

#### Development vs. Production Configs

The blueprint splits config into dev and publish variants. This is the key architectural pattern throughout:

- **Dev configs** (`babel.config.cjs`, `tsconfig.json`, `vite.config.mjs`) include macro evaluation, compat transforms, Vite/Embroider types, and test infrastructure
- **Publish configs** (`babel.publish.config.cjs`, `tsconfig.publish.json`, `rollup.config.mjs`) use minimal transforms and omit dev-only APIs

This split matters because macros should be evaluated by the consuming app (not baked in at publish time), and `lint:types` against the publish tsconfig catches accidental usage of Vite or Embroider internals in published code.

#### Vite Config (`vite.config.mjs`)

```javascript
import { defineConfig } from 'vite';
import { extensions, ember, classicEmberSupport } from '@embroider/vite';
import { babel } from '@rollup/plugin-babel';

// For scenario testing
const isCompat = Boolean(process.env.ENABLE_COMPAT_BUILD);

export default defineConfig({
  plugins: [
    ...(isCompat ? [classicEmberSupport()] : []),
    ember(),
    babel({
      babelHelpers: 'inline',
      extensions,
    }),
  ],
  build: {
    rollupOptions: {
      input: {
        tests: 'tests/index.html',
      },
    },
  },
});
```

#### Rollup Config (`rollup.config.mjs`)

```javascript
import { babel } from '@rollup/plugin-babel';
import { Addon } from '@embroider/addon-dev/rollup';
import { fileURLToPath } from 'node:url';
import { resolve, dirname } from 'node:path';

const addon = new Addon({
  srcDir: 'src',
  destDir: 'dist',
});

const rootDirectory = dirname(fileURLToPath(import.meta.url));
const babelConfig = resolve(rootDirectory, './babel.publish.config.cjs');
const tsConfig = resolve(rootDirectory, './tsconfig.publish.json');

export default {
  output: addon.output(),
  plugins: [
    addon.publicEntrypoints(['**/*.js', 'index.js', 'template-registry.js']),
    addon.appReexports([
      'components/**/*.js',
      'helpers/**/*.js',
      'modifiers/**/*.js',
      'services/**/*.js',
    ]),
    addon.dependencies(),
    babel({
      extensions: ['.js', '.gjs', '.ts', '.gts'],
      babelHelpers: 'bundled',
      configFile: babelConfig,
    }),
    addon.hbs(),
    addon.gjs(),
    // Emit .d.ts declaration files
    addon.declarations(
      'declarations',
      `npx @glint/ember-tsc -- --declaration --project ${tsConfig}`,
    ),
    addon.keepAssets(['**/*.css']),
    addon.clean(),
  ],
};
```

### TypeScript and Glint

TypeScript is opt-in via `--typescript`.

The blueprint uses two tsconfigs:

**`tsconfig.json`** (dev) -- includes `src/`, `tests/`, `demo-app/`, and `unpublished-development-types/`. Has Vite and Embroider types so your editor works.

```json
{
  "extends": "@ember/app-tsconfig",
  "include": [
    "src/**/*",
    "tests/**/*",
    "unpublished-development-types/**/*",
    "demo-app/**/*"
  ],
  "compilerOptions": {
    "rootDir": ".",
    "types": [
      "ember-source/types",
      "vite/client",
      "@embroider/core/virtual",
      "@glint/ember-tsc/types"
    ]
  }
}
```

**`tsconfig.publish.json`** -- only `src/` and dev types. No Vite or Embroider types, so `lint:types` catches accidental usage of dev-only APIs in published code.

```json
{
  "extends": "@ember/library-tsconfig",
  "include": ["./src/**/*", "./unpublished-development-types/**/*"],
  "compilerOptions": {
    "allowJs": true,
    "declarationDir": "declarations",
    "rootDir": "./src",
    "types": ["ember-source/types", "@glint/ember-tsc/types"]
  }
}
```

The Glint template registry (`src/template-registry.ts`) lets apps using loose mode (hbs files) consume your types. Not needed if your library only targets strict mode consumers.

### The Strict Resolver and `modules`

Both the test helper and the demo app use `ember-strict-application-resolver` instead of the classic Ember resolver. Instead of filesystem conventions, you explicitly register modules via a `modules` object. Each key must match a `./[type]/[name]` pattern (see [RFC 1132](https://rfcs.emberjs.com/id/1132-default-strict-resolver)):

```typescript
class MyApp extends EmberApp {
  modules = {
    './router': Router,                              // direct assignment
    './services/page-title': PageTitleService,        // explicit import
    ...import.meta.glob('./services/**/*', { eager: true }),  // bulk registration
    ...import.meta.glob('./templates/**/*', { eager: true }),
  };
}
```

You can register modules individually (useful for things from dependencies) or use `import.meta.glob` to sweep up everything in a directory. The glob approach is convenient but imports everything matching the pattern -- if you have non-service files in `services/`, they'll get pulled in too.

This pattern is used in two places:

- **Test helper** -- registers a minimal Router and optionally any services needed for tests
- **Demo app** -- registers the Router, templates, services, and anything else the demo needs

### Demo App

The blueprint includes a small demo application for manually testing your addon during development. Run `npm start` (or `pnpm start`) to launch Vite's dev server, which serves the root `index.html`:

**`index.html`**:
```html
<!doctype html>
<html lang="en-us">
<head>
  <meta charset="utf-8" />
  <title>Demo App</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <link rel="stylesheet" href="./demo-app/styles.css" />
</head>
<body>
  <script type="module">
    import { App } from './demo-app/app';
    App.create({})
  </script>
</body>
</html>
```

**`demo-app/app.gts`**:
```typescript
import EmberApp from 'ember-strict-application-resolver';
import EmberRouter from '@ember/routing/router';
import PageTitleService from 'ember-page-title/services/page-title';

class Router extends EmberRouter {
  location = 'history';
  rootURL = '/';
}

export class App extends EmberApp {
  modules = {
    './router': Router,
    './services/page-title': PageTitleService,
    ...import.meta.glob('./services/**/*', { eager: true }),
    ...import.meta.glob('./templates/**/*', { eager: true }),
  };
}

Router.map(function () {});
```

The demo app is a real Ember app -- it has routes, templates, and services -- but it boots directly via `ember-strict-application-resolver` with no ember-cli build step. Any addon code you want to exercise in the demo needs to be imported in the demo app's templates or registered in `modules`. The demo app's files are not published (they're not in the `files` array or `exports`).

### Testing

Tests also run entirely on Vite -- no ember-cli build pipeline, no webpack.

**`tests/test-helper.ts`**:
```typescript
import EmberApp from 'ember-strict-application-resolver';
import EmberRouter from '@ember/routing/router';
import * as QUnit from 'qunit';
import { setApplication } from '@ember/test-helpers';
import { setup } from 'qunit-dom';
import { start as qunitStart, setupEmberOnerrorValidation } from 'ember-qunit';
import { setTesting } from '@embroider/macros';

class Router extends EmberRouter {
  location = 'none';
  rootURL = '/';
}

class TestApp extends EmberApp {
  modules = {
    './router': Router,
    // add any custom services here
    // import.meta.glob('./services/*', { eager: true }),
  };
}

Router.map(function () {});

export function start() {
  setTesting(true);
  setApplication(
    TestApp.create({
      autoboot: false,
      rootElement: '#ember-testing',
    }),
  );
  setup(QUnit.assert);
  setupEmberOnerrorValidation();
  qunitStart();
}
```

**`tests/index.html`**:
```html
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <title><%= name %> Tests</title>
    <meta name="description" content="" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
  </head>
  <body>
    <div id="qunit"></div>
    <div id="qunit-fixture">
      <div id="ember-testing-container">
        <div id="ember-testing"></div>
      </div>
    </div>

    <script src="/testem.js" integrity="" data-embroider-ignore></script>
    <script type="module">
      import "ember-testing";
    </script>

    <script type="module">
      import { start } from "./test-helper.js";
      import.meta.glob("./**/*.{js,ts,gjs,gts}", { eager: true });
      start();
    </script>
  </body>
</html>
```

The test app is structurally the same as the demo app -- a minimal Ember app via `ember-strict-application-resolver` -- but configured for testing (`location = 'none'`, `autoboot: false`, `setTesting(true)`). Test discovery uses `import.meta.glob` in the HTML entry point. This is also a proof-of-concept for how future compat-less Ember apps could work.

#### Cross-Version Testing

The `.try.mjs` config defines scenarios for testing against multiple Ember versions:

```javascript
const compatFiles = {
  'ember-cli-build.cjs': `const EmberApp = require('ember-cli/lib/broccoli/ember-app');
const { compatBuild } = require('@embroider/compat');
module.exports = async function (defaults) {
  const { buildOnce } = await import('@embroider/vite');
  let app = new EmberApp(defaults);
  return compatBuild(app, buildOnce);
};`,
  'config/optional-features.json': JSON.stringify({
    'application-template-wrapper': false,
    'default-async-observers': true,
    'jquery-integration': false,
    'template-only-glimmer-components': true,
    'no-implicit-route-model': true,
  }),
};

const compatDeps = {
  '@embroider/compat': '^4.0.3',
  'ember-cli': '^5.12.0',
  'ember-auto-import': '^2.10.0',
  '@ember/optional-features': '^2.2.0',
};

export default {
  scenarios: [
    {
      name: 'ember-lts-5.8',
      npm: {
        devDependencies: {
          'ember-source': '~5.8.0',
          ...compatDeps,
        },
      },
      env: {
        ENABLE_COMPAT_BUILD: true,
      },
      files: compatFiles,
    },
    {
      name: 'ember-lts-5.12',
      npm: {
        devDependencies: {
          'ember-source': '~5.12.0',
          ...compatDeps,
        },
      },
      env: {
        ENABLE_COMPAT_BUILD: true,
      },
      files: compatFiles,
    },
    {
      name: 'ember-lts-6.4',
      npm: {
        devDependencies: {
          'ember-source': 'npm:ember-source@~6.4.0',
        },
      },
    },
    {
      name: 'ember-latest',
      npm: {
        devDependencies: {
          'ember-source': 'npm:ember-source@latest',
        },
      },
    },
    {
      name: 'ember-beta',
      npm: {
        devDependencies: {
          'ember-source': 'npm:ember-source@beta',
        },
      },
    },
    {
      name: 'ember-alpha',
      npm: {
        devDependencies: {
          'ember-source': 'npm:ember-source@alpha',
        },
      },
    },
  ],
};
```

Older Ember versions (5.x) need `@embroider/compat` and an `ember-cli-build.cjs` shim. Ember 6.4+ runs natively without compat mode.

### Babel Configs

**Dev (`babel.config.cjs`)** -- used by your editor and tests. Includes macro evaluation and compat transforms:

```javascript
/**
 * This babel.config is not used for publishing.
 * It's only for the local editing experience
 * (and linting)
 */
const { buildMacros } = require('@embroider/macros/babel');
const { babelCompatSupport, templateCompatSupport } = require('@embroider/compat/babel');

const macros = buildMacros();

// For scenario testing
const isCompat = Boolean(process.env.ENABLE_COMPAT_BUILD);

module.exports = {
  plugins: [
    ['@babel/plugin-transform-typescript', {
      allExtensions: true,
      allowDeclareFields: true,
      onlyRemoveTypeImports: true,
    }],
    ['babel-plugin-ember-template-compilation', {
      transforms: [
        ...(isCompat ? templateCompatSupport() : macros.templateMacros),
      ],
    }],
    ['module:decorator-transforms', {
      runtime: {
        import: require.resolve('decorator-transforms/runtime-esm'),
      },
    }],
    ...(isCompat ? babelCompatSupport() : macros.babelMacros),
  ],
  generatorOpts: {
    compact: false,
  },
};
```

**Publish (`babel.publish.config.cjs`)** -- minimal transforms. Macros are deliberately omitted; the consuming app evaluates them:

```javascript
/**
 * This babel.config is only used for publishing.
 *
 * For local dev experience, see the babel.config
 */
module.exports = {
  plugins: [
    ['@babel/plugin-transform-typescript', {
      allExtensions: true,
      allowDeclareFields: true,
      onlyRemoveTypeImports: true,
    }],
    ['babel-plugin-ember-template-compilation', {
      targetFormat: 'hbs',
      transforms: [],
    }],
    ['module:decorator-transforms', {
      runtime: {
        import: 'decorator-transforms/runtime-esm',
      },
    }],
  ],
  generatorOpts: {
    compact: false,
  },
};
```

### Linting

**ESLint** uses flat config (`eslint.config.mjs`) with `defineConfig` and `globalIgnores`:

```javascript
import babelParser from '@babel/eslint-parser/experimental-worker';
import js from '@eslint/js';
import { defineConfig, globalIgnores } from 'eslint/config';
import prettier from 'eslint-config-prettier';
import ember from 'eslint-plugin-ember/recommended';
import importPlugin from 'eslint-plugin-import';
import n from 'eslint-plugin-n';
import globals from 'globals';
import ts from 'typescript-eslint';

const esmParserOptions = {
  ecmaFeatures: { modules: true },
  ecmaVersion: 'latest',
};

const tsParserOptions = {
  projectService: true,
  tsconfigRootDir: import.meta.dirname,
};

export default defineConfig([
  globalIgnores(['dist/', 'dist-*/', 'declarations/', 'coverage/', '!**/.*']),
  js.configs.recommended,
  prettier,
  ember.configs.base,
  ember.configs.gjs,
  ember.configs.gts,
  {
    linterOptions: {
      reportUnusedDisableDirectives: 'error',
    },
  },
  {
    files: ['**/*.js'],
    languageOptions: {
      parser: babelParser,
    },
  },
  {
    files: ['**/*.{js,gjs}'],
    languageOptions: {
      parserOptions: esmParserOptions,
      globals: { ...globals.browser },
    },
  },
  {
    files: ['**/*.{ts,gts}'],
    languageOptions: {
      parser: ember.parser,
      parserOptions: tsParserOptions,
      globals: { ...globals.browser },
    },
    extends: [
      ...ts.configs.recommendedTypeChecked,
      { ...ts.configs.eslintRecommended, files: undefined },
      ember.configs.gts,
    ],
  },
  {
    files: ['src/**/*'],
    plugins: { import: importPlugin },
    rules: {
      'import/extensions': ['error', 'always', { ignorePackages: true }],
    },
  },
  // ... additional CJS/ESM node file configurations
]);
```

Type-aware linting via `projectService`, Ember .gjs/.gts support, and enforced file extensions in `src/` imports.

**Template linting** (`.template-lintrc.mjs`):
```javascript
export default {
  extends: 'recommended',
  checkHbsTemplateLiterals: false,
};
```

**Prettier** (`.prettierrc.mjs`):
```javascript
export default {
  plugins: ['prettier-plugin-ember-template-tag'],
  overrides: [
    {
      files: '*.{js,gjs,ts,gts,mjs,mts,cjs,cts}',
      options: {
        singleQuote: true,
        templateSingleQuote: false,
      },
    },
  ],
};
```

### CI/CD

The blueprint generates GitHub Actions workflows (shown here with pnpm; npm/yarn variants are also generated):

**CI workflow** (`ci.yml`):
- **Lint** -- runs `pnpm lint` (ESLint, Prettier, template-lint, type checking)
- **Test** -- runs `pnpm test`, then outputs a scenario matrix via `@embroider/try list`
- **Floating deps** -- installs without lockfile, runs tests to catch compatibility issues early
- **Try scenarios** -- matrix job that applies each `.try.mjs` scenario and runs tests against it

```yaml
try-scenarios:
  name: ${{ matrix.name }}
  runs-on: ubuntu-latest
  needs: "test"
  timeout-minutes: 10
  strategy:
    fail-fast: false
    matrix: ${{fromJson(needs.test.outputs.matrix)}}
  steps:
    - name: Apply Scenario
      run: pnpm dlx @embroider/try apply ${{ matrix.name }}
    - name: Install Dependencies
      run: pnpm install --no-lockfile
    - name: Run Tests
      run: pnpm test
      env: ${{ matrix.env }}
```

**Push dist workflow** (`push-dist.yml`) -- on push to main, builds the addon and pushes compiled assets to a `dist` branch for git-based consumption.

### V1 Compatibility

```javascript
// addon-main.cjs
'use strict';
const { addonV1Shim } = require('@embroider/addon-shim');
module.exports = addonV1Shim(__dirname);
```

This shim translates V2 package metadata into V1 build hooks so the addon works in classic ember-cli apps.

### Influence on Future App Blueprint

The test setup here is a proof-of-concept for future compat-less Ember apps:

- A minimal Ember app running on Vite with no webpack or `ember-cli-build.js`
- Bootstrap with just `EmberApp` and `EmberRouter` -- no complex build pipeline
- ES modules and `import.meta.glob` for module discovery instead of AMD/requirejs
- Direct framework API usage instead of ember-cli abstractions

This validates that Ember apps can run well on modern build tools, pointing toward simpler app blueprints in the future.

### Migration

Existing addons are unaffected. New addons get the new blueprint automatically. Existing addons can migrate by generating a new project and copying relevant files, or using `npx ember-cli@latest addon <name> --blueprint @ember/addon-blueprint`.

## How we teach this

### Documentation Updates

- Update the Ember Guides and CLI docs to reference the new blueprint
- The blueprint README covers customization, publishing, and multi-version support
- Provide migration guides for v1 and v2 addon authors

### Key Concepts for Addon Authors

#### `exports` and `imports`

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

**Recommendation**: Use `#src/*` imports for day-to-day development. The try-scenarios CI matrix will catch build/export issues by running against real builds. If you need to specifically test the published output, use consumer-style imports in a dedicated test file and run `npm run build` first.

#### Self-Imports

Self-imports (e.g. `import { x } from 'my-addon/foo'`) don't work during development in `src/` files because they resolve through `exports` to `dist/`, which doesn't exist until you build. Use relative imports in `src/`:

```javascript
// In src/ files:
import { myHelper } from './helpers/my-helper'; // yes
import { myHelper } from 'my-addon/helpers/my-helper'; // no
```

#### Dev vs. Publish Configs

| Purpose | Dev | Publish |
|---------|-----|---------|
| Babel | `babel.config.cjs` (macros, compat) | `babel.publish.config.cjs` (minimal) |
| TypeScript | `tsconfig.json` (all files, Vite types) | `tsconfig.publish.json` (src only) |
| Build | `vite.config.mjs` (HMR, tests) | `rollup.config.mjs` (tree-shaking) |

Macros are evaluated in dev for testing but left unevaluated in published output -- the consuming app handles them. The publish tsconfig omits Vite/Embroider types so `lint:types` catches accidental usage.

#### Monorepo Setup

The single-package default works for most addons. If you need a monorepo (complex integration testing, multiple related packages, full documentation app):

1. Generate addon: `npx ember-cli@latest addon my-addon --blueprint @ember/addon-blueprint --skip-git`
2. Remove generated test infrastructure
3. Generate test app: `npx ember-cli@latest app test-app --blueprint @ember/app-blueprint`
4. Set up workspace tooling (pnpm/yarn workspaces)
5. Install addon in test app

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
      "types": "./src/*.ts",
      "default": "./src/*.ts"
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

#### Publishing

1. Write code in `src/`, tests with `#src/*` imports
2. `npm run build` runs Rollup with publish configs, producing `dist/` and `declarations/`
3. `npm publish` ships only `files` from package.json
4. Consumers import via `exports`, not internal paths

### Resources

- [@ember/addon-blueprint README](https://github.com/emberjs/ember-addon-blueprint#readme)
- [Addon Author Guide](https://github.com/embroider-build/embroider/blob/main/docs/addon-author-guide.md)
- [Porting Addons to V2](https://github.com/embroider-build/embroider/blob/main/docs/porting-addons-to-v2.md)
- [Node.js Package Exports](https://nodejs.org/api/packages.html#exports)
- [Glint Documentation](https://typed-ember.gitbook.io/glint/)

## Drawbacks

- Some advanced use cases (monorepos, custom builds) need additional configuration.
- Addon authors unfamiliar with TypeScript/Glint face a learning curve, but JavaScript is fully supported.
- The blueprint is opinionated, but covers the vast majority of use cases.

## Alternatives

- Do nothing (keeps old blueprints, hinders modernization)
- Default to monorepo (too complex for most users)
- Provide multiple blueprints (maintenance burden, confusion)

## Unresolved questions

- How to best support advanced monorepo setups in the future.
- How to streamline migration for large, complex v1/v2 addons.
