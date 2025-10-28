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

This RFC proposes making [`@ember/addon-blueprint`](https://github.com/emberjs/ember-addon-blueprint) the default blueprint for new Ember addons, replacing the current v1 and v2 blueprints. The existing blueprints present significant technical challenges that impact developer productivity and ecosystem compatibility. The new blueprint addresses these issues through modern tooling, streamlined architecture, and comprehensive TypeScript integration based on extensive community feedback and production usage patterns.

## Motivation

The current default blueprints have architectural limitations that affect addon development:

**V1 addons** require rebuilding by every consuming application, creating performance overhead and compatibility constraints. This approach differs from modern JavaScript packaging principles and creates coupling between addon build processes and consuming applications.

**The original V2 blueprint** addressed the build performance issues but introduced additional complexity through mandatory monorepo structure and required manual configuration. Additionally, it lacked integration with modern Ember patterns, TypeScript tooling, and development workflows that have become common in the ecosystem.

The `@ember/addon-blueprint` addresses these technical problems through several architectural improvements: it defaults to single-package structure (appropriate for most addon use cases), integrates Glint for template type safety, implements modern Ember patterns throughout, and provides tooling configuration. This blueprint incorporates lessons learned from production addon development and addresses the issues identified by the community.

## Detailed design

### Definitions

**V2 Addon**: An addon published in the V2 package format as defined by RFC 507, with `ember-addon.version: 2` in package.json.

**Glint-enabled**: A package that includes TypeScript declaration files for templates and components, leveraging Volar-based language server plugins for superior IDE integration.

**Single-package addon**: An addon that contains its own test suite within the same package, as opposed to a monorepo structure with separate test applications.

**Blueprint**: A code generation template used by ember-cli to scaffold new projects or files.

**Vite-first development**: A development workflow that uses Vite as the primary build tool for both development and testing, eliminating traditional webpack-based tooling.

### Blueprint Structure and Tooling Choices

The blueprint generates a modern project structure:

```
my-addon/
├── .github/
│   └── workflows/
│       ├── ci.yml                    # Comprehensive CI/CD pipeline
│       └── push-dist.yml             # Automated dist branch publishing
├── src/                              # Source code (published)
│   ├── index.js                      # Main entry point
│   └── template-registry.ts          # Glint type registry
├── tests/                            # Test files
│   ├── index.html                    # Test runner page
│   └── test-helper.ts                # Test setup and configuration
├── demo-app/                         # Demo application for showcasing addon
│   ├── app.gts                       # Demo app entry point
│   ├── styles.css                    # Demo app styles
│   └── templates/                    # Demo app templates
├── unpublished-development-types/    # Development-only types
│   └── index.d.ts                    # Local development type augmentations
├── dist/                             # Built output (gitignored, published)
├── declarations/                     # TypeScript declarations (gitignored, published)
├── package.json                      # Package configuration with modern exports
├── rollup.config.mjs                 # Production build configuration
├── vite.config.mjs                   # Development build configuration
├── tsconfig.json                     # Development TypeScript config
├── tsconfig.publish.json             # Publishing TypeScript config
├── babel.config.cjs                  # Development Babel config
├── babel.publish.config.cjs          # Publishing Babel config
├── eslint.config.mjs                 # Modern ESLint flat config
├── .prettierrc.cjs                   # Code formatting configuration
├── .template-lintrc.cjs              # Template linting rules
├── testem.cjs                        # Test runner configuration
├── .try.mjs                          # Ember version compatibility scenarios
├── .npmrc                            # Package manager configuration
├── .editorconfig                     # Editor consistency rules
├── .env.development                  # Development environment variables
├── README.md                         # Documentation template
├── CONTRIBUTING.md                   # Contribution guidelines
├── LICENSE.md                        # MIT license
└── addon-main.cjs                    # V1 compatibility shim
```

The blueprint architecture addresses several specific technical requirements:

**Single-package structure by default** - Analysis of existing addons shows that monorepo complexity is not needed for most use cases, while adding maintenance overhead. The single-package approach simplifies the development workflow without restricting advanced users who require monorepo setups.

**Structured source organization** - The `src/` directory contains publishable code, `tests/` contains the test suite, and `unpublished-development-types/` provides development-only type augmentations that don't pollute the published package. This separation ensures clean package boundaries and prevents accidental inclusion of development-only code.

**Glint integration** - Template type safety is implemented through Glint with Volar-based TypeScript server plugins, providing improved IDE integration compared to previous approaches. This addresses template type validation in Ember applications.

**Modern Ember patterns throughout** - The blueprint uses native classes, decorators, and strict mode, eliminating legacy object model patterns that create maintenance burden and limit performance optimizations.

**Dual build system approach** - Vite provides improved development experience through fast rebuilds and hot module replacement, while Rollup generates optimized production output with effective tree-shaking and bundle optimization. Tests execute entirely through Vite, reducing the complexity associated with `ember-auto-import` and webpack-based tooling.

#### Implications for Future App Blueprint Architecture

The testing architecture implemented in this addon blueprint serves as a technical proof-of-concept for future compat-less Ember application blueprints.

The test environment demonstrates a minimal Ember application running entirely on Vite without webpack or ember-cli-build.js dependencies. This architecture eliminates complex build pipeline abstractions while maintaining full framework compatibility, suggesting viability for production application development.

The `tests/test-helper.js` implementation creates Ember applications using only `EmberApp`, `Resolver`, and `EmberRouter` - a significantly simplified bootstrap process compared to traditional ember-cli approaches. This pattern demonstrates how future applications could reduce initialization complexity while preserving framework functionality.

The architecture utilizes ES modules and standard import semantics throughout, replacing the AMD/requirejs patterns prevalent in traditional Ember applications. The test discovery mechanism employs Vite's `import.meta.glob`, which could extend to application development for route and component discovery through standard module resolution rather than custom file-system scanning.

The successful operation of this testing approach demonstrates that Ember applications can work effectively with modern build tools, providing a foundation for simplified application blueprints in future releases.

### Package Configuration

> [!NOTE]
> This is overview, and exact contents can change as the needs of the community and capabilities of the ecosystem change. We're always striving to simplify the setup, while enabling the most power for the 80% use case, while not restricting anyone who wants to do more complex things.

#### package.json Structure

The generated `package.json` follows modern NPM conventions with several key design decisions:

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
    "dist"
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

The package configuration implements several important architectural patterns:

The exports map utilizes conditional exports to ensure proper resolution of both TypeScript declarations and JavaScript modules across all environments. Import maps (`#src/*`) enable clean internal imports for test and demo application code, though these cannot be used within `src/` files due to Rollup's transformation limitations during the build process.

The published package includes only essential runtime files (`dist`, `declarations`, `addon-main.cjs`) to minimize package size, while the `ember-addon.version: 2` declaration ensures proper V2 addon recognition by tooling and consuming applications.

#### Development vs. Production Configuration Split

The blueprint maintains separate configurations for development and production builds:

**Development Configuration (`vite.config.mjs`)**:
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

Enables fast development builds with HMR, compatibility mode for older Ember versions, and direct test execution through Vite.

**Production Configuration (`rollup.config.mjs`)**:
```javascript
import { babel } from '@rollup/plugin-babel';
import { Addon } from '@embroider/addon-dev/rollup';

const addon = new Addon({
  srcDir: 'src',
  destDir: 'dist',
});

export default {
  output: addon.output(),
  plugins: [
    addon.publicEntrypoints(['**/*.js', 'index.js', 'template-registry.js']),
    addon.appReexports([
      // (instance) initializers are omitted from this list, because not many libraries provide them.
      // There is also an upcoming RFC on this topic.
      'components/**/*.js',
      'helpers/**/*.js',
      'modifiers/**/*.js',
      'services/**/*.js',
    ]),
    addon.dependencies(),
    babel({
      extensions: ['.js', '.gjs', '.ts', '.gts'],
      babelHelpers: 'bundled',
      configFile: './babel.publish.config.cjs',
    }),
    addon.hbs(),
    addon.gjs(),
    // Emit .d.ts declaration files
    addon.declarations('declarations', 'npx ember-tsc --declaration --project ./tsconfig.publish.json'),
    addon.keepAssets(['**/*.css']),
    addon.clean(),
  ],
};
```

Ensures optimized output with tree-shaking, automatic TypeScript declaration generation (when using `--typescript`), V2 package compliance, and proper asset handling.

### TypeScript and Glint Integration

TypeScript is opt-in via `--typescript` when generating a new addon.

#### Dual TypeScript Configuration Strategy

The blueprint employs separate TypeScript configurations for development and publishing:

**Development Configuration (`tsconfig.json`)**:

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

**Publishing Configuration (`tsconfig.publish.json`)**:

Notably, vite and embroider types are not present. `lint:types` uses this config to help catch accidental usage of vite or embroider features in code that would be published to NPM.

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

The development config includes test files and development types so your IDE works properly, while the publishing config generates clean declaration files with the right module structure.

#### Glint Template Type Safety

The blueprint includes comprehensive Glint setup for template type safety. However, note that the template-registry is not needed at all if a library does not want to support loose mode (hbs files).

**Template Registry (`src/template-registry.ts`)**:
```typescript
// Easily allow apps, which are not yet using strict mode templates, to consume your Glint types, by importing this file.
// Add all your components, helpers and modifiers to the template registry here, so apps don't have to do this.
// See https://typed-ember.gitbook.io/glint/environments/ember/authoring-addons

// import type MyComponent from './components/my-component';

// Uncomment this once entries have been added! 👇
// export default interface Registry {
//   MyComponent: typeof MyComponent
// }
```

**Development Type Augmentations (`unpublished-development-types/index.d.ts`)**:

This file provides a space for development-only type augmentations that won't be included in the published package. It's initially empty but available for local development type definitions.

Provides consumer type safety, development-only types that don't pollute the published package, and full template import support.

### Testing Architecture

#### Vite-Based Test Execution

The blueprint implements a revolutionary testing approach that runs entirely on Vite:

**Test Helper (`tests/test-helper.ts`)**:
```typescript
import EmberApp from 'ember-strict-application-resolver';
import EmberRouter from '@ember/routing/router';
import * as QUnit from 'qunit';
import { setApplication } from '@ember/test-helpers';
import { setup } from 'qunit-dom';
import { start as qunitStart, setupEmberOnerrorValidation } from 'ember-qunit';

class Router extends EmberRouter {
  location = 'none';
  rootURL = '/';
}

class TestApp extends EmberApp {
  modules = {
    './router': { default: Router },
    // add any custom services here
    // import.meta.glob('./services/*', { eager: true }),
  };
}

Router.map(function () {});

export function start() {
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

**Test Runner (`tests/index.html`)**:
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

This architecture eliminates the ember-cli build pipeline entirely by creating Ember applications directly through public APIs, utilizing `import.meta.glob` for test discovery, and avoiding webpack dependencies completely. This approach reduces complexity while maintaining full Ember compatibility.

#### Cross-Version Compatibility Testing

The blueprint includes comprehensive compatibility testing via ember-try scenarios:

**Try Configuration (`.try.mjs`)**:
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
    // Additional scenarios for 5.12, 6.4, latest, beta, alpha
  ],
};
```

Tests against all supported Ember LTS versions, beta/alpha releases, with automatic compat builds for older versions and modern feature flags enabled.

### Build System Architecture

#### Dual Build Approach Rationale

Uses two build systems:

**Vite for development**: Instant startup with native ES modules, HMR, modern debugging tools, and integrated test execution.

**Rollup for production**: Optimized output with tree-shaking, bundle splitting, asset processing, and TypeScript declaration generation.

#### Babel Configuration Strategy

The blueprint maintains separate Babel configurations for development and production:

**Development Babel (`babel.config.cjs`)**:

Like the `tsconfig.json`, this is the configuration used by editors and tests.

```javascript
const { buildMacros } = require('@embroider/macros/babel');
const { babelCompatSupport, templateCompatSupport } = require('@embroider/compat/babel');

/**
 * In testing, we need to evaluate @embroider/macros
 * in the publish config we omit this, because it's the consuming app's responsibility to evaluate the macros. 
 */
const macros = buildMacros();
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

**Production Babel (`babel.publish.config.cjs`)**:

> [!IMPORTANT]
> @embroider/macros is deliberately omitted from this config, because it's the consuming app's responsibility to evaluate macros.

```javascript
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

The configuration separation addresses distinct requirements: development configurations include macro evaluation and compatibility transforms for comprehensive testing capabilities, while production configurations use minimal transforms and delegate macro processing to consuming applications, ensuring proper build-time optimization boundaries.

### Influence on Future V2 App Blueprint

The test architecture in this addon blueprint serves as a prototype for how we envision a future compat-less `@ember/app-blueprint` should work:

#### Architectural Principles Demonstrated

**Vite-First Architecture**: The addon's test setup demonstrates a minimal Ember application running entirely on Vite without any webpack or ember-cli-build.js complexity. This same pattern would be ideal for v2 apps - direct Vite integration with Ember resolver and routing eliminates the need for complex build pipeline abstractions.

**Minimal Application Bootstrap**: The `tests/test-helper.js` creates a bare-bones Ember app using just `EmberApp`, `Resolver`, and `EmberRouter`. This approach eliminates the traditional ember-cli build pipeline and shows how future apps could bootstrap with much less ceremony. The pattern of directly importing and configuring Ember's core classes provides a blueprint for simpler app initialization.

**Modern Module Resolution**: The test setup uses ES modules and modern imports throughout, avoiding the complex AMD/requirejs patterns of traditional Ember apps. Future v2 apps should follow this same module pattern, using standard `import` statements and module resolution rather than the custom loader.js system.

**Direct Framework Integration**: Rather than going through ember-cli's abstraction layer, the addon tests interact directly with Ember's APIs. This demonstrates the cleaner architectural approach we want for v2 apps - direct framework usage without heavy tooling intermediation. The test setup shows how to create and configure an Ember application using only public APIs.

**Zero-Config Test Execution**: The test runner uses Vite's `import.meta.glob` for automatic test discovery, eliminating the need for complex test configuration. This pattern could extend to app development, where route and component discovery happens through standard module resolution rather than custom file-system scanning.

The success of this addon testing approach validates that Ember applications can run efficiently with modern build tools, paving the way for a simpler, faster app blueprint that matches this architecture.

### CI/CD Setup

The blueprint includes a comprehensive GitHub Actions workflow with several jobs, each serving a specific purpose:

#### Lint Job Analysis
```yaml
lint:
  name: "Lints"
  runs-on: ubuntu-latest
  timeout-minutes: 10
  steps:
    - uses: actions/checkout@v4
    - uses: pnpm/action-setup@v4  # if using pnpm
    - uses: actions/setup-node@v4
      with:
        node-version: 22
        cache: pnpm
    - name: Install Dependencies
      run: pnpm install --frozen-lockfile
    - name: Lint
      run: pnpm lint
```

Runs ESLint, Prettier, and template-lint with caching for consistent code quality across contributors.

#### Test Job with Matrix Generation
```yaml
test:
  name: "Tests"
  runs-on: ubuntu-latest
  timeout-minutes: 10
  outputs:
    matrix: ${{ steps.set-matrix.outputs.matrix }}
  steps:
    # ... setup steps
    - name: Run Tests
      run: pnpm test
    - id: set-matrix
      run: |
        echo "matrix=$(pnpm -s dlx @embroider/try list)" >> $GITHUB_OUTPUT
```

Runs the test suite and generates the scenario matrix for cross-version compatibility testing.

#### Floating Dependencies Job
```yaml
floating:
  name: "Floating Dependencies"
  runs-on: ubuntu-latest
  timeout-minutes: 10
  steps:
    # ... setup steps
    - name: Install Dependencies
      run: pnpm install --no-lockfile
    - name: Run Tests
      run: pnpm test
```

Tests against latest dependency versions without lockfile constraints to catch compatibility issues early.

#### Try Scenarios Matrix Job
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
    # We setup the scenario before running `pnpm install`, so this is fast.
    # No double-install like there is with ember-try.
    # This tool can also be used with non-ember projects.
    - name: Apply Scenario
      run: pnpm dlx @embroider/try apply ${{ matrix.name }}
    - name: Install Dependencies
      run: pnpm install --no-lockfile
    - name: Run Tests
      run: pnpm test
      env: ${{ matrix.env }}
```

Uses `@embroider/try` to test against multiple Ember versions (LTS, stable, beta, alpha) ensuring compatibility across the supported ecosystem.

#### Push Dist Workflow

```yaml
name: Push dist
on:
  push:
    branches: [main, master]
jobs:
  push-dist:
    name: Push dist
    permissions:
      contents: write
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      # ... build and publish to dist branch
```

Automatically builds and pushes compiled assets to a `dist` branch for Git-based consumption during development.

### Linting and Code Quality Configuration

#### ESLint Configuration
The blueprint uses modern ESLint flat configuration (`eslint.config.mjs`):

```javascript
import babelParser from '@babel/eslint-parser';
import js from '@eslint/js';
import prettier from 'eslint-config-prettier';
import ember from 'eslint-plugin-ember/recommended';
import importPlugin from 'eslint-plugin-import';
import n from 'eslint-plugin-n';
import globals from 'globals';
import ts from 'typescript-eslint';

const config = [
  js.configs.recommended,
  prettier,
  ember.configs.base,
  ember.configs.gjs,
  ember.configs.gts,
  
  // File-specific configurations for different contexts
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
    },
    extends: [...ts.configs.recommendedTypeChecked, ember.configs.gts],
  },
  // ... additional configurations
];

export default ts.config(...config);
```

This uses ESLint's new flat configuration format (which is faster and clearer), includes full TypeScript support with type-aware linting, has comprehensive Ember-specific rules including .gjs/.gts support, uses different rules for different contexts (browser vs. Node.js), and validates your imports to make sure you're using relative paths correctly.

#### Template Linting
```javascript
// .template-lintrc.cjs
module.exports = {
  extends: 'recommended',
  checkHbsTemplateLiterals: false,
};
```

The template linting configuration uses the recommended rule set but disables checking of template literals, focusing on .hbs files and template imports.

#### Prettier Configuration
```javascript
// .prettierrc.cjs
module.exports = {
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

This configuration ensures consistent formatting across all JavaScript/TypeScript files while properly handling .gjs/.gts template tags.

### Package Management Configuration

#### NPM Configuration (`.npmrc`)
```
# we don't want addons to be bad citizens of the ecosystem
auto-install-peers=false

# we want true isolation, if a dependency is not declared, we want an error
resolve-peers-from-workspace-root=false
```

The NPM configuration enforces explicit dependency declaration by disabling auto-install-peers and prevents workspace root resolution to ensure complete package isolation. These constraints guarantee proper dependency boundaries and maintain NPM ecosystem compatibility standards.

### V1 Compatibility Strategy

#### Compatibility Shim (`addon-main.cjs`)
```javascript
'use strict';
const { addonV1Shim } = require('@embroider/addon-shim');
module.exports = addonV1Shim(__dirname);
```

This minimal shim enables V2 addons to work in V1 (classic ember-cli) environments by providing the traditional addon API that ember-cli expects.

**How It Works:**
- The shim translates V2 package metadata into V1 build hooks
- Static assets and app reexports are handled through traditional treeFor* methods
- The V2 package's dist/ output is mapped to the appropriate V1 trees

### Technical Decision Analysis

#### Single Package Architecture
Analysis of the Ember addon ecosystem shows that monorepo structures are used by fewer than 20% of published addons, while adding complexity overhead for all users. Single packages provide better maintainability through reduced configuration surface area, simplified dependency management, and streamlined publishing workflows. Advanced users requiring monorepo architectures can utilize the blueprint as a foundation while implementing custom test harnesses.

#### Glint with Volar Integration
Template type safety addresses an important gap in the Ember development experience, helping catch errors at compile time that would otherwise appear at runtime. Glint provides this through template analysis, while Volar-based TypeScript server plugins provide IDE integration without custom language server implementations. This approach delivers type safety benefits to both TypeScript and JavaScript projects through type inference and validation.

#### Dual Build System Strategy
Performance testing shows that Vite provides significantly faster development rebuilds compared to traditional webpack-based approaches, while Rollup generates well-optimized production output through effective tree-shaking algorithms and bundle optimization. The dual approach reduces the performance vs. output quality trade-offs present in single build system architectures, allowing optimization of each phase of the development lifecycle.

### Migration and Compatibility

Existing addons are unaffected. Authors can opt into the new blueprint with `npx ember-cli@latest addon <name> --blueprint @ember/addon-blueprint`. 

**Migration:** New addons use the new blueprint immediately. Existing addons can migrate by generating a new project and copying relevant files.


## How we teach this

### Core Documentation Updates

- The Ember Guides and CLI documentation will be updated to reference the new blueprint.
- The blueprint README includes detailed instructions for customization, publishing, and supporting multiple Ember versions.
- Migration guides will be provided for authors of v1 and v2 addons.

### Library Development Concepts

#### Understanding `package.json#exports` and `package.json#imports`

**`package.json#exports`** defines the public API of your addon - what consuming applications can import:

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

- **"."** - The main entry point when someone imports your addon by name
- **"./*.css"** - Allows importing CSS files from your addon 
- **"./*"** - Allows importing any other file from your dist directory
- **"types"** - Points to TypeScript declaration files for proper IDE support
- **"default"** - Points to the actual JavaScript implementation

**`package.json#imports`** provides internal shortcuts for development:

```json
{
  "imports": {
    "#src/*": "./src/*"
  }
}
```

- **#src/*** - Enables clean imports like `import { helper } from '#src/helpers/my-helper'` in tests
- **Tests and demo app only** - Can only be used in test files and demo app, NOT in `src/` files
- **Rollup limitation** - Rollup doesn't transform `#src/*` imports to relative paths during the build process
- **Development convenience** - Avoids relative paths in tests and eliminates need for constant rebuilding during test development

**Important**: Files in `src/` must use standard relative imports (`./`, `../`) or imports from `node_modules`. The `#src/*` imports are only for development convenience in test files.

#### Self-Imports Limitation

**Self-imports are not available during development**. Self-imports would resolve through `package.json#exports`, but during development, the `dist/` directory (referenced in exports) doesn't exist yet - development works directly with source files in `src/`.

```javascript
// ❌ This won't work in src/ files during development
import { myHelper } from 'my-addon/helpers/my-helper';

// ✅ Use relative imports instead
import { myHelper } from './helpers/my-helper';
```

Self-imports only work after building and publishing your addon, when consuming applications import from the published `dist/` files.

#### Development vs. Production Configuration Strategy

The blueprint uses dual configurations to optimize both development experience and published output:

**Development Configuration (Local Development & Testing)**:
- `babel.config.cjs` - Includes macro evaluation and compatibility transforms
- `tsconfig.json` - Includes test files, demo app, and development types
- `vite.config.mjs` - Optimized for fast rebuilds with HMR

**Production Configuration (Publishing)**:
- `babel.publish.config.cjs` - Minimal transforms, no macro evaluation (consuming app's responsibility)
- `tsconfig.publish.json` - Only source files, generates clean declarations
- `rollup.config.mjs` - Optimized output with tree-shaking and proper exports

**Why This Separation Matters**:
- **Development**: Includes all necessary transforms and types for productive local development
- **Publishing**: Generates clean, minimal output that integrates well with consuming applications
- **Macro Handling**: Development evaluates macros for testing; production leaves them for the consuming app to handle
- **Type Safety**: Publishing config catches accidental usage of development-only APIs (Vite, Embroider internals)

#### Rollup Build Process

The Rollup configuration transforms your `src/` directory into the `dist/` directory that gets published:

```javascript
addon.publicEntrypoints(['**/*.js', 'index.js', 'template-registry.js'])
```

- **publicEntrypoints** - Defines what files consumers can import (matches `package.json#exports`)
- **appReexports** - Automatically makes components/helpers/etc. available in consuming apps
- **declarations** - Generates TypeScript declarations alongside JavaScript
- **keepAssets** - Preserves CSS and other static files

The build process:
1. Transpiles TypeScript/templates using Babel (publish config)
2. Generates TypeScript declarations using `ember-tsc`
3. Creates optimized JavaScript in `dist/` matching your `src/` structure
4. Preserves CSS and other assets

#### When to Use Monorepo Structure

**Use the default single-package structure when**:
- Building a typical addon with components, helpers, or services
- Your addon doesn't need complex integration testing
- You want minimal setup and maintenance overhead
- Your tests can adequately cover addon functionality in isolation

**Consider a monorepo structure when**:
- **Complex Integration Testing**: Your addon needs to be tested within a real application context (routing, complex data flows, etc.)
- **Multiple Related Packages**: You're building an addon ecosystem with multiple related packages
- **Advanced Testing Scenarios**: You need to test against multiple Ember app configurations or build setups
- **Documentation Apps**: You want a full application for showcasing your addon's capabilities

**Setting up a Monorepo**:
1. Generate your addon: `npx ember-cli@latest addon my-addon --blueprint @ember/addon-blueprint --skip-git`
2. Remove the generated test infrastructure
3. Generate a test app: `npx ember-cli@latest app test-app --blueprint @ember/app-blueprint`
4. Configure your monorepo tool (pnpm workspaces, yarn workspaces, etc.)
5. Install your addon in the test app and write integration tests

This approach provides the benefits of the modern addon blueprint while giving you a full application environment for comprehensive testing.

#### Package Publishing Workflow

**Development Workflow**:
1. Write code in `src/`
2. Write tests using `#src/*` imports for convenience
3. Run tests with `npm test` (uses development configs)
4. Use demo app for manual testing and documentation

**Publishing Workflow**:
1. `npm run build` uses Rollup with production configs
2. Generates `dist/` and `declarations/` directories
3. `npm publish` only publishes files listed in `package.json#files`
4. Consumers import from your `package.json#exports`, not internal paths

#### TypeScript Integration

**When using `--typescript`**:
- All configurations include TypeScript support
- Glint provides template type checking
- Development includes comprehensive types (Vite, Embroider, etc.)
- Publishing generates clean declaration files for consumers
- `lint:types` script catches development-only API usage in published code

**JavaScript-only projects**:
- All TypeScript-specific features are conditional
- Still benefits from modern build pipeline and tooling
- Can opt into TypeScript later with minimal changes

### Key Resources

- [@ember/addon-blueprint README](https://github.com/emberjs/ember-addon-blueprint#readme)
- [Addon Author Guide](https://github.com/embroider-build/embroider/blob/main/docs/addon-author-guide.md) 
- [Porting Addons to V2](https://github.com/embroider-build/embroider/blob/main/docs/porting-addons-to-v2.md)
- [Node.js Package Exports Documentation](https://nodejs.org/api/packages.html#exports)
- [Glint Documentation](https://typed-ember.gitbook.io/glint/)

These resources will be integrated into the official Ember Guides to provide comprehensive addon development guidance.

## Drawbacks

- Some advanced use cases (e.g., monorepos, custom build setups) may require additional manual configuration.
- Addon authors unfamiliar with TypeScript or Glint may face a learning curve, but JavaScript is still supported.
- The blueprint is opinionated, which may not suit all workflows, but it is designed to cover the vast majority of use cases.

## Alternatives

- Do nothing (keep the old blueprints, but this would hinder ecosystem modernization).
- Make monorepo the default (adds complexity for most users).
- Provide multiple blueprints (increases maintenance burden and confusion).

## Unresolved questions

- How to best support advanced monorepo setups in the future.
- How to streamline migration for large, complex v1/v2 addons.
