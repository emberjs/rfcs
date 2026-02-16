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

The ESLint configuration would include the e18e plugin:

```js
import e18e from '@e18e/eslint-plugin';

export default [
  // ... other configs
  e18e.configs.recommended,
  // ... project-specific overrides
];
```

### Phased Rollout:

The plugin could be introduced with warnings initially, then gradually moved to errors as the ecosystem adopts the patterns. This follows the same pattern used for other linting rules in Ember CLI.

## How we teach this

### Documentation Updates:

1. **Ember CLI Guides**: Add a section explaining the @e18e/eslint-plugin and its purpose in the default linting setup
2. **Linting Guide**: Update the linting documentation to mention the e18e rules alongside existing ESLint and template-lint documentation
3. **Migration Guide**: For existing projects, provide instructions on how to add the plugin manually

### Teaching Strategy:

The @e18e/eslint-plugin should be taught as a natural extension of Ember's existing linting infrastructure, similar to how `eslint-plugin-ember` is currently positioned. Key points:

- The plugin helps developers write modern, performant JavaScript code
- It's part of Ember's commitment to shipping modern applications
- The rules are auto-fixable in many cases, reducing friction
- Developers can disable specific rules if they conflict with project requirements

### Communication:

- Blog post announcing the addition, explaining the benefits and showing examples
- Include in Ember CLI release notes
- Highlight the auto-fix capabilities to reduce the learning burden

### For New Users:

New users would naturally encounter these rules as they develop, with helpful error messages and suggestions. The experience would be similar to learning ESLint rules for the first time - the linter guides them toward better patterns.

### For Existing Users:

Existing projects would not be affected unless they upgrade their blueprints or manually add the plugin. Documentation would help teams evaluate whether to adopt it in existing codebases.

## Drawbacks

- **Additional dependency**: Adds another package to the dependency tree, though the size impact should be minimal
- **More linting messages**: Developers may see additional linting warnings/errors, especially in projects with older code patterns
- **Learning curve**: Developers need to understand another set of linting rules
- **Rule conflicts**: Potential for conflicts with existing project-specific ESLint rules or team preferences
- **Breaking changes**: As the @e18e/eslint-plugin evolves, rule updates could introduce new warnings/errors in projects
- **Ecosystem maturity**: The plugin is relatively new compared to established tools like eslint-plugin-ember

## Alternatives

### Do Nothing
Continue with the current default linting setup, letting developers add @e18e/eslint-plugin manually if desired. This avoids the drawbacks but also means new projects miss out on modern best practices by default.

### Add as Optional Addon
Create an optional addon or blueprint flag that allows developers to opt-in during project creation. This reduces friction for those who don't want the additional rules but may lead to fragmentation in the ecosystem.

### Use Individual Rules
Instead of the full plugin, cherry-pick specific rules that align most closely with Ember's goals. This provides more control but requires ongoing maintenance to evaluate new rules.

### Wait for Broader Adoption
Delay adding the plugin to blueprints until it sees wider adoption in the JavaScript ecosystem. This reduces risk but means Ember projects lag behind modern best practices.

### Create Ember-Specific Plugin
Develop an Ember-specific linting plugin that incorporates similar modernization rules tailored to the Ember ecosystem. This would provide more control but requires significant development and maintenance effort.

## Unresolved questions

- What version of @e18e/eslint-plugin should be included initially?
- Should any specific rules be disabled or configured differently for Ember projects?
- Should the rules be set to "warn" or "error" by default?
- How should this interact with existing TypeScript ESLint configurations in apps that use TypeScript?
- Should there be a way to opt-out during project generation?
