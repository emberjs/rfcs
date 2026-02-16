---
stage: accepted
start-date: 2026-02-16T00:00:00.000Z
release-date: # In format YYYY-MM-DDT00:00:00.000Z
release-versions:
teams:
  - cli
prs:
  accepted: # Fill this in with the URL for the Proposal RFC PR
project-link:
suite: 
---

# Add @e18e/eslint-plugin to Blueprints' Lint Configs

## Summary

Add https://github.com/e18e/eslint-plugin to the app and addon blueprints' ESLint configurations using the recommended rules.

## Motivation

The @e18e/eslint-plugin provides modern JavaScript/TypeScript best practices focused on code modernization, performance improvements, and recommended module replacements. By including this plugin in the default blueprints, new Ember applications and addons will benefit from:

- **Code Modernization**: Automatic suggestions to use newer JavaScript APIs (e.g., `Array.prototype.at()`, `Array.prototype.includes()`, etc.) that improve code readability and often performance
- **Performance Improvements**: Detection of patterns that can be optimized for better runtime performance
- **Module Replacements**: Recommendations for community-preferred alternatives to popular libraries, focused on bundle size and performance
- **Consistency**: All new Ember projects start with the same modern coding standards, helping teams maintain consistency across the ecosystem

This aligns with Ember's commitment to shipping apps with modern JavaScript patterns and helping developers write high-quality, performant code by default.

## Detailed design

1. Add `@e18e/eslint-plugin` as a dev dependency to the app blueprint
2. Add `@e18e/eslint-plugin` as a dev dependency to the addon blueprint
3. Update the ESLint configuration in both blueprints to include the recommended rules from the plugin

### Implementation Steps:

**For the app blueprint:**
- Add `"@e18e/eslint-plugin"` to the `devDependencies` in `blueprints/app/files/package.json` using the latest stable version at the time of implementation
- Update the ESLint config (typically in the blueprint's ESLint configuration file) to extend or include the `@e18e/eslint-plugin/recommended` configuration

**For the addon blueprint:**
- Similar to the app blueprint, add the dependency and configure ESLint to use the plugin's recommended rules
- This would be done in a manner similar to how `eslint-plugin-ember` is currently added to addon blueprints

### Configuration Example:

The ESLint configuration would include the e18e plugin. Additionally, the e18e plugin includes rules that can lint `package.json` files (e.g., `ban-dependencies`), so we should configure JSON linting support:

```js
import e18e from '@e18e/eslint-plugin';
import json from '@eslint/json';

export default [
  // Enable e18e rules for all JavaScript/TypeScript files
  e18e.configs.recommended,
  
  // Enable JSON linting with e18e rules for package.json
  {
    files: ['package.json'],
    language: 'json/json',
    ...json.configs.recommended,
    plugins: {
      e18e,
    },
    rules: {
      // e18e rules that apply to package.json
      ...e18e.configs.recommended.rules,
    }
  },
  
  // ... other project-specific configs
];
```

## How we teach this

### Documentation Updates:

Documentation updates for this feature would require new pages in the ember-learn/guides-source repository, as there is currently no comprehensive explanation of Ember's approach to linting. These documentation pages would be valuable to have but are outside the scope of this RFC:

1. **Ember CLI Guides**: Would benefit from a section explaining the @e18e/eslint-plugin and its purpose in the default linting setup
2. **Linting Guide**: Could include documentation about the e18e rules alongside existing ESLint and template-lint documentation
3. **Migration Guide**: For existing projects, could provide instructions on how to add the plugin manually

### Teaching Strategy:

The @e18e/eslint-plugin should be taught as a natural extension of Ember's existing linting infrastructure, similar to how `eslint-plugin-ember` is currently positioned. The recommended set of ESLint rules provides a good signal to users starting new projects about how they should maintain their apps and libraries. Key points:

- The plugin helps developers write modern, performant JavaScript code
- It's part of Ember's commitment to shipping modern applications
- The rules are auto-fixable in many cases, reducing friction
- Developers can disable specific rules if they conflict with project requirements

### For New Users:

New users would naturally encounter these rules as they develop, with helpful error messages and suggestions. The experience would be similar to learning ESLint rules for the first time - the linter guides them toward better patterns.

### For Existing Users:

Existing projects would not be affected unless they upgrade their blueprints or manually add the plugin. Users' processes would be unchanged from normal—most developers already customize their lint configurations and would need to evaluate any changes carefully as they normally do when updating dependencies or adopting new tooling.

## Drawbacks

- **Additional dependency**: Adds another package to the dependency tree, though the size impact should be minimal
- **More linting messages**: Developers may see additional linting warnings/errors, especially in projects with older code patterns
- **Learning curve**: Developers need to understand another set of linting rules

## Alternatives

### Do Nothing
Continue with the current default linting setup, letting developers add @e18e/eslint-plugin manually if desired. This avoids the drawbacks but also means new projects miss out on modern best practices by default.

## Unresolved questions

None. This RFC proposes using the latest version of @e18e/eslint-plugin with its recommended configuration and default settings. Developers can manually remove the plugin by deleting the relevant lines from their configuration if they prefer not to use it.
