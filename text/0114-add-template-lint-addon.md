---
start-date: 2018-01-04T00:00:00.000Z
release-date:
release-versions: 
  0: F
  1: I
  2: X
  3: M
  4: E

teams: 
  - cli
prs:
  accepted: https://github.com/ember-cli/rfcs/pull/114
project-link: 
stage: discontinued
---

# Summary

Add https://github.com/rwjblue/ember-cli-template-lint as a default addon for the app and addon blueprints using the recommended rules.

# Motivation

Linting and security in templates would help not only individual developers write better apps with better accessibility and security, but would also help teams to be on the same page and stick to a handful of standards.

# Detailed design

1. Move ember-cli-template-lint to the ember-cli org (better for contributing and getting work off one person, @rwjblue)
2. Add the dependency to the app blueprint here: https://github.com/ember-cli/ember-cli/blob/master/blueprints/app/files/package.json#L19
3. Also add it to the addon blueprint, like the eslint addon here: https://github.com/ember-cli/ember-cli/blob/master/blueprints/addon/index.js#L66

# How We Teach This

The same way that we teach ESLint being on by default.

# Drawbacks

- More chatter in the terminal.
- An additional dependency.
- Recommended rules might not be good for everyone.. but that same issue probably exists with ESLint.

# Alternatives

Do nothing and have people write sub par template code.

# Unresolved questions

None
