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

This RFC proposes to make [`@ember/addon-blueprint`](https://github.com/emberjs/ember-addon-blueprint) the default blueprint for new Ember addons, replacing the previous v1 and v2 blueprints. The new blueprint is the result of extensive community feedback, real-world usage, and a focus on modern JavaScript, TypeScript, and Ember best practices. It is designed to provide a streamlined, ergonomic, and future-proof starting point for addon authors.

## Motivation

The previous default blueprints (classic v1 and the original v2 via `@embroider/addon-blueprint`) have served the community well, but both have significant drawbacks:

- **V1 Addons**: Built by consuming apps on every build, leading to slow builds and complex compatibility issues.
- **Original V2 Addon Blueprint**: Improved build performance by building at publish time, but required significant manual setup, was monorepo-oriented by default, and lacked ergonomic defaults for modern Ember and TypeScript usage.

The new `@ember/addon-blueprint` addresses these issues by:

- Providing a single-package, non-monorepo structure by default, reducing complexity for most addon authors.
- Including first-class Glint support with Volar-based tsserver plugins, providing the most TypeScript-like experience possible for Ember templates and components.
- Using modern JavaScript and Ember idioms, including native classes, decorators, and strict mode.
- Integrating with the latest Ember testing and linting tools out of the box.
- Reducing boilerplate and cognitive overhead for new addon authors.

## Detailed design

### Definitions

**V2 Addon**: An addon published in the V2 package format as defined by RFC 507, with `ember.version: 2` in package.json.

**Glint-enabled**: A package that includes TypeScript declaration files for templates and components, leveraging Volar-based language server plugins for superior IDE integration.

**Single-package addon**: An addon that contains its own test suite within the same package, as opposed to a monorepo structure with separate test applications.

**Blueprint**: A code generation template used by ember-cli to scaffold new projects or files.

**Vite-first development**: A development workflow that uses Vite as the primary build tool for both development and testing, eliminating traditional webpack-based tooling.

### Blueprint Structure and Tooling Choices

The new blueprint generates a well-organized project structure that follows modern Ember and JavaScript conventions:

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

The new blueprint makes several key decisions:

- **Single-package, non-monorepo by default**: Most addons do not require a monorepo structure. The blueprint generates a single package with integrated testing, but can be adapted for monorepo setups by ignoring the test infrastructure and creating a separate test application.

- **Source Organization**: The `src/` directory contains all publishable code, while `tests/` contains the test suite. The `unpublished-development-types/` directory provides a space for development-only type augmentations that won't be included in the published package.

- **Glint-enabled**: All source files include Glint configuration by default, leveraging Volar-based tsserver plugins for superior template type checking and IDE integration. The `template-registry.ts` file provides type safety for consuming applications. This was not available in previous blueprints when opting into TypeScript via `--typescript`.

- **Modern Ember Patterns**: Uses native classes, decorators, and strict mode. No legacy Ember object model or classic patterns.

- **Dual Build Systems**: Vite for development (fast rebuilds, HMR) and Rollup for publishing (optimized output, tree-shaking). Each has its own configuration file and Babel setup.

- **Testing**: Integrates with `@ember/test-helpers` and `qunit` for modern testing. Unlike previous blueprints, testing runs entirely through Vite (accessible via the `/tests` URL and CLI), eliminating the need for `ember-auto-import` and webpack for test execution.

#### Influence on Future V2 App Blueprint

The test architecture in this addon blueprint serves as a prototype for how we envision a future compat-less `@ember/app-blueprint` should work:

- **Vite-First Architecture**: The addon's test setup demonstrates a minimal Ember application running entirely on Vite without any webpack or ember-cli-build.js complexity. This same pattern would be ideal for v2 apps - direct Vite integration with Ember resolver and routing eliminates the need for complex build pipeline abstractions.

- **Minimal Application Bootstrap**: The `tests/test-helper.js` creates a bare-bones Ember app using just `EmberApp`, `Resolver`, and `EmberRouter`. This approach eliminates the traditional ember-cli build pipeline and shows how future apps could bootstrap with much less ceremony. The pattern of directly importing and configuring Ember's core classes provides a blueprint for simpler app initialization.

- **Modern Module Resolution**: The test setup uses ES modules and modern imports throughout, avoiding the complex AMD/requirejs patterns of traditional Ember apps. Future v2 apps should follow this same module pattern, using standard `import` statements and module resolution rather than the custom loader.js system.

- **Direct Framework Integration**: Rather than going through ember-cli's abstraction layer, the addon tests interact directly with Ember's APIs. This demonstrates the cleaner architectural approach we want for v2 apps - direct framework usage without heavy tooling intermediation. The test setup shows how to create and configure an Ember application using only public APIs.

- **Zero-Config Test Execution**: The test runner uses Vite's `import.meta.glob` for automatic test discovery, eliminating the need for complex test configuration. This pattern could extend to app development, where route and component discovery happens through standard module resolution rather than custom file-system scanning.

The success of this addon testing approach validates that Ember applications can run efficiently with modern build tools, paving the way for a simpler, faster app blueprint that matches this architecture.

### Package Configuration Analysis

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
    "./*": {
      "types": "./declarations/*.d.ts",
      "default": "./dist/*.js"
    },
    "./addon-main.js": "./addon-main.cjs"
  },
  "typesVersions": {
    "*": {
      "*": ["declarations/*"]
    }
  }
}
```

**Key Design Decisions:**

- **Modern Exports Map**: Uses conditional exports to provide both TypeScript declarations and JavaScript modules, ensuring proper resolution in all environments.
- **Import Maps**: The `#src/*` import map enables clean internal imports without relative paths -- these are meant for use in tests only, and should not be left in files that are built for npm. When published, importers of your library will resolve via `package.json#exports`, rather than `package.json#imports`'s `#src`. Using `#src/` means that you don't need to repeatedly build, or run your library's build in watch mode while working on tests.
- **Minimal Published Files**: Only publishes essential runtime files (`dist`, `declarations`, `addon-main.cjs`), keeping package size minimal.
- **V2 Package Metadata**: Declares `ember.version: 2` to indicate V2 package format compliance.

#### Development vs. Production Configuration Split

The blueprint maintains separate configurations for development and production builds:

**Development Configuration (`vite.config.mjs`)**:
```javascript
import { defineConfig } from 'vite';
import { extensions, ember, classicEmberSupport } from '@embroider/vite';
import { babel } from '@rollup/plugin-babel';

// Used for try scenarios where you would want to test ember-source versions < 6.3
const isCompat = Boolean(process.env.ENABLE_COMPAT_BUILD);

export default defineConfig({
  resolve: {
    alias: [
      {
        // this enables self-imports within the addon's src files, which will work when built 
        // via rollup, due to self-imports being a native node-resolve feature 
        // (and something we've been used to with v1 addons)
        find: '<%= the library name %>',
        replacement: `${__dirname}/src`,
      },
    ],
  },
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

This configuration enables:
- **Fast Development Builds**: Vite's native ES module support provides instant rebuilds
- **Hot Module Replacement**: Changes reflect immediately without full page reloads
- **Compatibility Mode**: Conditional support for older Ember versions via environment variables
- **Test Integration**: Direct test execution through Vite's build system

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
    // This requires glint@v2/alpha
    addon.declarations('declarations', 'npx glint --declaration --project ./tsconfig.publish.json'),
    addon.keepAssets(['**/*.css']),
    addon.clean(),
  ],
};
```

This configuration ensures:
- **Optimized Output**: Tree-shaking and dead code elimination
- **Type Declaration Generation**: Automatic TypeScript declaration file creation -- though note that while TypeScript is supported is always optional and opt in (via `--typescript`)
- **V2 Package Compliance**: Proper app reexports and public entrypoint definition
- **Asset Handling**: CSS and other static assets are properly included

### TypeScript and Glint Integration

TypeScript is opt-in via `--typescript` when generating a new addon.

#### Dual TypeScript Configuration Strategy

The blueprint employs separate TypeScript configurations for development and publishing:

**Development Configuration (`tsconfig.json`)**:

The exact configurations here may change as glint @ v2 makes more progress -- for example, there is desire to bundle ember-loose and ember-template-imports environments in to `@glint/core`, so we have 2 less packages when working with Glint.

```json
{
  "extends": "@ember/app-tsconfig",
  "glint": {
    "environment": ["ember-loose", "ember-template-imports"]
  },
  "include": ["src/**/*", "tests/**/*", "unpublished-development-types/**/*"],
  "compilerOptions": {
    "rootDir": ".",
    "types": ["ember-source/types", "vite/client", "@embroider/core/virtual"]
  }
}
```

**Publishing Configuration (`tsconfig.publish.json`)**:

Notably, vite and embroider types are not present.

`lint:types` uses this config to help catch accidental usage of vite or embroider features in code that would be published to NPM.

```json
{
  "extends": "@ember/library-tsconfig",
  "include": ["./src/**/*", "./unpublished-development-types/**/*"],
  "glint": {
    "environment": ["ember-loose", "ember-template-imports"]
  },
  "compilerOptions": {
    "allowJs": true,
    "declarationDir": "declarations",
    "rootDir": "./src",
    "types": ["ember-source/types"]
  }
}
```

**Rationale for Dual Configurations:**
- **Development Flexibility**: Includes test files and development-only types for comprehensive IDE support
- **Clean Published Types**: Publishing config generates minimal, focused declaration files
- **Proper Module Structure**: `rootDir` alignment ensures declaration file structure matches runtime module structure

#### Glint Template Type Safety

The blueprint includes comprehensive Glint setup for template type safety. However, note that the template-registry is not needed at all if a library does not want to support loose mode (hbs files).

**Template Registry (`src/template-registry.ts`)**:
```typescript
// Easily allow apps, which are not yet using strict mode templates, to consume your Glint types
// by importing this file. Add all your components, helpers and modifiers to the template registry
// here, so apps don't have to do this.

// import type MyComponent from './components/my-component';

// export default interface Registry {
//   MyComponent: typeof MyComponent
// }
```

**Development Type Augmentations (`unpublished-development-types/index.d.ts`)**:

This can be omitted if a library author doesn't need to support loose mode / hbs files.

```typescript
// Add any types here that you need for local development only.
// These will *not* be published as part of your addon, so be careful that your published 
// code does not rely on them!

import '@glint/environment-ember-loose';
import '@glint/environment-ember-template-imports';
```

This setup provides:
- **Consumer Type Safety**: Apps consuming the addon get proper template type checking
- **Development-Only Types**: Local development can use additional type definitions without polluting the published package
- **Template Import Support**: Full support for template imports and strict mode templates

### Testing Architecture

#### Vite-Based Test Execution

The blueprint implements a revolutionary testing approach that runs entirely on Vite:

**Test Helper (`tests/test-helper.ts`)**:
```typescript
import EmberApp from '@ember/application';
import Resolver from 'ember-resolver';
import EmberRouter from '@ember/routing/router';
import * as QUnit from 'qunit';
import { setApplication } from '@ember/test-helpers';
import { setup } from 'qunit-dom';
import { start as qunitStart, setupEmberOnerrorValidation } from 'ember-qunit';

class Router extends EmberRouter {
  location = 'none';
  rootURL = '/';
}

const registry = {
  'test-app/router': { default: Router },
  // add any custom services here
  // example:
  //   'test-app/services/store': { default: Store },
  // and/or initializers:
  //   'test-app/instance-initializers/foo': { default: Foo }
  // 
  // NOTE: when using (instance)initializers, you'll need https://github.com/ember-cli/ember-load-initializers/
  //       and to call loadInitializers(TestApp, 'test-app', registry);
}

class TestApp extends EmberApp {
  modulePrefix = 'test-app';
  Resolver = Resolver.withModules(registry);
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

**Key Innovations:**
- **No ember-cli-build.js**: Tests run without traditional Ember CLI build pipeline, `ember-cli` is not even in the package.json.
- **Direct Ember App Creation**: Creates minimal Ember application using core _public_ APIs
- **ES Module Test Discovery**: Uses Vite's `import.meta.glob` for automatic test file discovery
- **Zero Webpack Dependencies**: Eliminates `ember-auto-import` and webpack from test execution

#### Cross-Version Compatibility Testing

The blueprint includes comprehensive compatibility testing via ember-try scenarios:

**Try Configuration (`.try.mjs`)**:
```javascript
const compatFiles = {
  'ember-cli-build.js': `const EmberApp = require('ember-cli/lib/broccoli/ember-app');
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

This provides:
- **LTS Version Support**: Tests against all supported Ember LTS versions
- **Future Compatibility**: Tests against beta and alpha releases
- **Compatibility Mode**: Automatic fallback to compat builds for older versions
- **Modern Feature Flags**: Enables modern Ember features for optimal performance

### Build System Architecture

#### Dual Build Approach Rationale

The blueprint employs two distinct build systems for different purposes:

**Vite for Development**:
- **Instant Startup**: Native ES module support eliminates bundling during development
- **Hot Module Replacement**: Changes reflect immediately without full rebuilds
- **Modern Development Experience**: Source maps, error overlay, and debugging tools
- **Test Integration**: Seamless test execution within the same build system

**Rollup for Production**:
- **Optimized Output**: Tree-shaking and dead code elimination
- **Bundle Splitting**: Optimal chunk creation for different consumption patterns
- **Asset Processing**: Proper handling of CSS, templates, and static assets
- **Declaration Generation**: TypeScript declaration file creation

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

**Configuration Differences:**
- **Development**: Includes macro support and compatibility transforms for older Ember versions
- **Production**: Minimal transforms focusing on TypeScript removal and template compilation
- **Runtime Imports**: Development uses require.resolve for local development, production uses published packages

### Influence on Future V2 App Blueprint

The test architecture in this addon blueprint serves as a prototype for how we envision a future compat-less `@ember/app-blueprint` should work:

#### Architectural Principles Demonstrated

**Vite-First Architecture**: The addon's test setup demonstrates a minimal Ember application running entirely on Vite without any webpack or ember-cli-build.js complexity. This same pattern would be ideal for v2 apps - direct Vite integration with Ember resolver and routing eliminates the need for complex build pipeline abstractions.

**Minimal Application Bootstrap**: The `tests/test-helper.js` creates a bare-bones Ember app using just `EmberApp`, `Resolver`, and `EmberRouter`. This approach eliminates the traditional ember-cli build pipeline and shows how future apps could bootstrap with much less ceremony. The pattern of directly importing and configuring Ember's core classes provides a blueprint for simpler app initialization.

**Modern Module Resolution**: The test setup uses ES modules and modern imports throughout, avoiding the complex AMD/requirejs patterns of traditional Ember apps. Future v2 apps should follow this same module pattern, using standard `import` statements and module resolution rather than the custom loader.js system.

**Direct Framework Integration**: Rather than going through ember-cli's abstraction layer, the addon tests interact directly with Ember's APIs. This demonstrates the cleaner architectural approach we want for v2 apps - direct framework usage without heavy tooling intermediation. The test setup shows how to create and configure an Ember application using only public APIs.

**Zero-Config Test Execution**: The test runner uses Vite's `import.meta.glob` for automatic test discovery, eliminating the need for complex test configuration. This pattern could extend to app development, where route and component discovery happens through standard module resolution rather than custom file-system scanning.

The success of this addon testing approach validates that Ember applications can run efficiently with modern build tools, paving the way for a simpler, faster app blueprint that matches this architecture.

### CI/CD Workflow Analysis and Defense

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
        node-version: 18
        cache: pnpm
    - name: Install Dependencies
      run: pnpm install --frozen-lockfile
    - name: Lint
      run: pnpm lint
```

The lint job runs ESLint, Prettier, and template-lint checks to ensure code quality and consistent formatting. This catches style issues early and maintains consistency across contributors. The job uses caching to improve performance and includes both JavaScript/TypeScript and Handlebars template linting.

**Defense**: Linting is critical for maintaining code quality in a distributed development environment. By running linting in CI, we ensure that all contributors follow the same code standards, reducing review friction and maintaining consistency.

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

This job runs the addon's test suite and generates a matrix of scenarios for compatibility testing. It ensures the addon works correctly and prepares the scenario matrix for broader compatibility testing. The job outputs the matrix for use by the try-scenarios job.

**Defense**: The dual purpose of this job (testing and matrix generation) optimizes CI efficiency. We test the addon in its primary configuration while simultaneously preparing for cross-version compatibility testing.

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

This job tests the addon against the latest available versions of all dependencies (without lockfile constraints). It catches potential issues with newer dependency versions early and ensures the addon remains compatible as the ecosystem evolves.

**Defense**: Floating dependencies testing is crucial for maintaining forward compatibility. Since addons are consumed by apps with varying dependency versions, testing without lockfile constraints helps identify potential breaking changes before they affect users.

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

This job uses `@embroider/try` to test against multiple Ember versions and dependency combinations defined in `.try.mjs`. It ensures addon compatibility across the supported Ember ecosystem, including LTS versions, latest stable, beta, and alpha releases. Each scenario can include different build modes (like compat builds for older Ember versions).

**Defense**: Comprehensive cross-version testing is essential for addon ecosystem health. By testing against multiple Ember versions, we catch breaking changes early and ensure addons work across the entire supported version range.

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

This workflow automatically builds and pushes compiled assets to a `dist` branch, enabling easy cross-repository testing and consumption of the addon from Git repositories.

**Defense**: The dist branch workflow enables modern development practices where addons can be consumed directly from Git repositories during development. This is particularly valuable for testing pre-release versions and coordinating changes across multiple repositories.

### Linting and Code Quality Configuration

#### ESLint Configuration Analysis
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

**Key Features:**
- **Modern Flat Config**: Uses ESLint's new flat configuration format for better performance and clearer semantics
- **TypeScript Integration**: Full TypeScript support with type-aware linting rules
- **Ember-Specific Rules**: Comprehensive Ember linting including .gjs/.gts support
- **Context-Aware Rules**: Different rules for different file types (browser vs. Node.js)
- **Import Validation**: Ensures proper import/export usage and relative path requirements

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

**Design Rationale:**
- **Explicit Dependencies**: Disabling auto-install-peers forces explicit declaration of all dependencies
- **True Isolation**: Preventing workspace root resolution ensures package isolation and prevents hidden dependencies
- **Ecosystem Citizenship**: These settings encourage proper dependency declaration, making addons better NPM citizens

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

### Rationale and Defense of Choices

#### Single Package Architecture
Most addons are single packages that don't require monorepo complexity. The blueprint generates a single package with integrated testing, but can be adapted for monorepo setups by ignoring the test infrastructure and creating a separate test application using `@ember/app-blueprint`.

**Benefits:**
- **Reduced Complexity**: Single package structure is easier to understand and maintain
- **Faster Setup**: No need to coordinate multiple packages during development
- **Simpler Publishing**: Single package publishing is more straightforward than monorepo coordination
- **Better IDE Support**: IDEs can better understand and index single-package structures

**Monorepo Migration Path**: For teams currently using monorepo setups from the old `@embroider/addon-blueprint`, the new pattern involves generating your addon with `@ember/addon-blueprint` (ignoring its test setup) and then using a separate test application via `@ember/app-blueprint` (or other app blueprint). This maintains the monorepo benefits while providing cleaner separation of concerns.

#### Glint with Volar Integration
The Ember ecosystem is moving towards TypeScript, and Glint provides the best template type-checking experience. The Volar-based tsserver plugins deliver a native TypeScript-like IDE experience that was previously unavailable in addon blueprints, enabling superior autocomplete, error detection, and refactoring capabilities in templates.

**Technical Advantages:**
- **Template Type Safety**: Full type checking for Handlebars templates
- **IDE Integration**: Native TypeScript language server support
- **Incremental Adoption**: Works with both TypeScript and JavaScript
- **Consumer Benefits**: Apps consuming the addon get automatic template type checking

#### Vite Development Architecture
Vite provides the fastest possible development experience with instant rebuilds and hot reloading, significantly improving addon development productivity. The Vite-based test architecture eliminates traditional build pipeline complexity while maintaining full Ember compatibility.

**Performance Benefits:**
- **Instant Startup**: Native ES module support eliminates bundling delays
- **Hot Module Replacement**: Changes reflect immediately without full page reloads
- **Parallel Processing**: Vite's architecture enables better CPU utilization
- **Modern Development**: Source maps, error overlay, and debugging tools

#### Rollup Production Builds
Rollup generates optimized, tree-shakeable output that consuming applications can efficiently bundle. The separation between development (Vite) and production (Rollup) builds allows each tool to excel at its intended purpose.

**Optimization Benefits:**
- **Tree Shaking**: Unused code is eliminated from the final bundle
- **Bundle Splitting**: Optimal chunk creation for different consumption patterns
- **Asset Optimization**: Proper handling of CSS, templates, and static assets
- **Declaration Generation**: Clean TypeScript declaration file creation

### Migration and Compatibility

Existing addons are not affected by this change. Authors can opt into the new blueprint already by running `npx ember-cli@latest addon <name> --blueprint @ember/app-blueprint`. The blueprint is versioned and locked to the Ember CLI release, ensuring reproducibility.

**Migration Strategies:**

**Greenfield Addons**: New addons can immediately use the new blueprint for optimal development experience

For existing addons there is no need to migrate, however if, after trying the new blueprint, folks like the experience better, they can:
- For V1 addons, follow the "Porting Addons to V2" guide, or generate a new project and copy relevant files into it.
- For V2 addons, generate a new project and copy relevant files into it.


## How we teach this

- The Ember Guides and CLI documentation will be updated to reference the new blueprint.
- The blueprint README includes detailed instructions for customization, publishing, and supporting multiple Ember versions.
- Migration guides will be provided for authors of v1 and v2 addons.
- Key resources (so far):
  - [@ember/addon-blueprint README](https://github.com/emberjs/ember-addon-blueprint#readme)
  - [Addon Author Guide](https://github.com/embroider-build/embroider/blob/main/docs/addon-author-guide.md)
  - [Porting Addons to V2](https://github.com/embroider-build/embroider/blob/main/docs/porting-addons-to-v2.md)

These will, of course, need to be added to the guides website as well.

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
