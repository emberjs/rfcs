- Start Date: 2017-02-02
- RFC PR: (leave this empty)
- Ember CLI Issue: (leave this empty)

# Summary

Enable Ember CLI users to opt into using yarn for packagement management.

# Motivation

Ember CLI currently uses the npm command line tool to install dependencies when you run `ember install` or `ember new`/`ember addon`. However, several problems usually arise from npm's semantics.
Dependency resolution and install times can be significant enough to disrupt workflows, as well as offline support, non-deterministic, non-flat dependency graphs.

Yarn was introduced to the JavaScript community with the intent to provide a better developer experience in those areas:
- Faster installs
- Offline support
- Deterministic dependency graphs
- Lockfile semantics

While Ember CLI users can currently use Yarn to manage their dependencies, Ember CLI will use the npm client internally when running the above mentioned commands. This document intends to give the users the ability to tell Ember CLI to use Yarn internally, for a consistent experience.

# Detailed design

- rewrite ember install to use yarn if yarn.lock exists
- rewrite ember init/new/addon to use yarn if --yarn is used

# How We Teach This

- Guides
- `ember help`

# Drawbacks

None.

# Alternatives

Do nothing.

# Unresolved questions

None.
