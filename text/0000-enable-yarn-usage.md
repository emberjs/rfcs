- Start Date: 2017-02-02
- RFC PR: (leave this empty)
- Ember CLI Issue: (leave this empty)

# Summary

Enable Ember CLI users to opt into using yarn for packagement management.

# Motivation

Yarn provides a better developer experience:
- Faster installs
- Offline support
- Deterministic dependency graphs
- Lockfile semantics

# Detailed design

- rewrite ember install to use yarn if yarn.lock exists
- rewrite ember init/new/addon to use yarn if --yarn is used

# How We Teach This

- Guides
- `ember help`

# Drawbacks

None.

# Alternatives

Continue with npm.

# Unresolved questions

None.
