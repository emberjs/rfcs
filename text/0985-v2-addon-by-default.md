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

The new blueprint makes several key decisions:

- **Single-package, non-monorepo by default**: Most addons do not require a monorepo structure. The blueprint generates a single package, but can be extended to monorepo setups if needed.
- **Glint-enabled**: All source files include Glint configuration by default, leveraging Volar-based tsserver plugins for superior template type checking and IDE integration. This was not available in previous blueprints when opting into TypeScript via `--typescript`.
- **Modern Ember Patterns**: Uses native classes, decorators, and strict mode. No legacy Ember object model or classic patterns.
- **Testing**: Integrates with `@ember/test-helpers` and `qunit` for modern testing. Unlike previous blueprints, testing runs entirely through Vite (accessible via the `/tests` URL and CLI), eliminating the need for `ember-auto-import` and webpack for test execution.
- **Linting and Formatting**: Pre-configures `eslint`, `prettier`, and Ember-specific linting rules for code consistency.
- **Documentation**: Includes a README template and guides for publishing, versioning, and supporting multiple Ember versions.
- **CI/CD**: Provides a GitHub Actions workflow for testing, linting, and publishing, reducing setup time for new projects.
- **Peer Dependencies**: Clearly specifies peer dependencies for Ember and related packages, avoiding version conflicts.

#### Rationale and Defense of Choices

- **Non-monorepo default**: Most addons are single packages. Monorepo setups add unnecessary complexity for the majority of users. However, for those who need monorepo structures (like the old `@embroider/addon-blueprint` pattern), you can simply not use the testing capabilities of the new blueprint and create a separate test application using `@ember/app-blueprint`. This provides the same monorepo benefits while keeping the blueprint focused on the common single-package case.
- **Glint with Volar**: The Ember ecosystem is moving towards TypeScript, and Glint provides the best template type-checking experience. The Volar-based tsserver plugins deliver a native TypeScript-like IDE experience that was previously unavailable in addon blueprints, enabling superior autocomplete, error detection, and refactoring capabilities in templates.
- **Modern idioms**: Encourages best practices and prepares addons for future Ember releases.
- **Pre-configured tooling**: Reduces time spent on setup and avoids common pitfalls, especially for new authors.
- **CI/CD**: Ensures that all addons start with a robust, modern workflow, improving quality and reliability.

#### CI/CD Workflow Job Analysis and Defense

The blueprint includes a comprehensive GitHub Actions workflow (`.github/workflows/ci.yml`) with several jobs, each serving a specific purpose:

- **Lint Job**: Runs ESLint, Prettier, and template-lint checks to ensure code quality and consistent formatting. This catches style issues early and maintains consistency across contributors. The job uses caching to improve performance and includes both JavaScript/TypeScript and Handlebars template linting.

- **Test Job (Matrix Setup)**: Runs the addon's test suite and generates a matrix of scenarios for compatibility testing. This ensures the addon works correctly and prepares the scenario matrix for broader compatibility testing. The job outputs the matrix for use by the try-scenarios job.

- **Floating Dependencies Job**: Tests the addon against the latest available versions of all dependencies (without lockfile constraints). This catches potential issues with newer dependency versions early and ensures the addon remains compatible as the ecosystem evolves.

- **Try Scenarios Job (Matrix)**: Uses `@embroider/try` to test against multiple Ember versions and dependency combinations defined in `.try.mjs`. This ensures addon compatibility across the supported Ember ecosystem, including LTS versions, latest stable, beta, and alpha releases. Each scenario can include different build modes (like compat builds for older Ember versions).

**Additional CI Features:**
- **Push Dist Workflow**: Automatically builds and pushes compiled assets to a `dist` branch, enabling easy cross-repository testing and consumption of the addon from Git repositories.
- **Concurrency Control**: Prevents duplicate CI runs for the same branch/PR, saving resources and providing faster feedback.
- **Timeout Protection**: Each job has reasonable timeouts to prevent runaway processes from consuming excessive CI resources.

**Rationale**: This comprehensive CI approach ensures addon quality, compatibility, and reliability across the entire Ember ecosystem. The matrix testing strategy catches breaking changes early, while the floating dependencies job helps maintain forward compatibility. The push-dist workflow enables modern development workflows where addons can be consumed directly from Git. This level of CI sophistication was not standard in previous blueprints and often required significant manual setup.

#### Build and Development Tooling Choices

Beyond CI, the blueprint makes several key tooling decisions:

- **Vite for Development**: Uses Vite for fast development builds and testing, providing hot module replacement and modern development experience. The `vite.config.mjs` includes Ember-specific plugins and compat mode support.

- **Rollup for Publishing**: Uses Rollup via `@embroider/addon-dev` for production builds, ensuring optimal bundle sizes and proper ESM output. The `rollup.config.mjs` includes TypeScript declaration generation when TypeScript is enabled.

- **Dual TypeScript Configs**: Separates development (`tsconfig.json`) and publishing (`tsconfig.publish.json`) configurations, allowing different settings for local development versus published types.

- **Modern Test Architecture**: Uses a custom test application setup in `tests/test-helper.js` that provides a minimal Ember app for testing addon functionality without the overhead of a full Ember app structure.

- **Try Scenarios**: The `.try.mjs` file defines comprehensive compatibility testing across multiple Ember versions, including both modern Vite builds and legacy compat builds for older Ember versions.

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
