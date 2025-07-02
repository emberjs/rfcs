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
**Defense of Tooling Choices:**
- **Vite**: Provides the fastest possible development experience with instant rebuilds and hot reloading, significantly improving addon development productivity.
- **Rollup**: Generates optimized, tree-shakeable output that consuming applications can efficiently bundle.
- **Dual configs**: Allows development flexibility while ensuring clean, minimal published types.
- **Comprehensive testing**: The try scenarios cover the full range of supported Ember versions, catching compatibility issues before they reach users.

### Migration and Compatibility

- Existing addons are not affected.
- Authors can opt into the new blueprint by running `ember addon <name>` with the latest Ember CLI.
- The blueprint is versioned and locked to the Ember CLI release, ensuring reproducibility.
- **Monorepo Migration**: For teams currently using monorepo setups from the old `@embroider/addon-blueprint`, the new pattern involves generating your addon with `@ember/addon-blueprint` (ignoring its test setup) and then using a separate test application via `@ember/app-blueprint` (or other app blueprint). 

## How we teach this

- The Ember Guides and CLI documentation will be updated to reference the new blueprint.
- The blueprint README includes detailed instructions for customization, publishing, and supporting multiple Ember versions.
- Migration guides will be provided for authors of v1 and v2 addons.
- Key resources (so far):
  - [@ember/addon-blueprint README](https://github.com/emberjs/ember-addon-blueprint#readme)
  - [Addon Author Guide](https://github.com/embroider-build/embroider/blob/main/docs/addon-author-guide.md)
  - [Porting Addons to V2](https://github.com/embroider-build/embroider/blob/main/docs/porting-addons-to-v2.md)

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
