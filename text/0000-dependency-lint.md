- Start Date: 2017-03-06/2019-03-14
- Relevant Team(s): Ember CLI
- RFC PR: (after opening the RFC PR, update this with a link to it and update the file name)
- Tracking: (leave this empty)

# Summary

Add [ember-cli-dependency-lint](https://github.com/salsify/ember-cli-dependency-lint) to the default app blueprint.

# Motivation

When installing dependencies for an Ember app, it's possible to end up with multiple copies of a given addon if it's
transitively needed by other addons with conflicting version requirements. When Ember CLI includes that addon into a
built app, only one version will ultimately make it in, meaning that code written expecting a different version may not
work properly. Today this situation is hard to detect without manual inspection, and may arise any time you add or
update a dependency.

# Detailed design

Further details are available in [ember-cli-dependency-lint's README](https://github.com/salsify/ember-cli-dependency-lint),
but at a high level it validates for each addon in the build that only one version is present, and then exposes any
violations of that rule in two ways:
 - as failing lint tests to alert the user to the problem during the normal development flow
 - via an explicit `ember dependency-lint` command that facilitates faster cycles when troubleshooting a dependency issue

Users may override the "only one version" rule on a case by case basis by specifying a semver expression.

# How We Teach This

Many users today haven't deeply considered the underlying problem that ember-cli-dependency-lint detects. As Ember
Weekend called out, "it helps you debug an issue you probably didn't know you had".

The problem itself has proven pretty straightforward to grasp once folks are presented with it, but understanding what
action to take when it rears its head requires a more subtle understanding of how npm packages and dependency resolution
work than many people seem to have. A quick primer in the Ember CLI documentation may help with this (and actually
might be a nice add regardless of this RFC), but the exact strategy for this is called out as an Unresolved Question
below.

_A note on the name:_ I realized right after publishing that ember-cli-dependency-checker is also a thing, and that I'd
missed that before. There's also ember-cli-version-checker, though that's not an addon and not in the default blueprint.
If there are concerns about confusion around the name, I'd welcome suggestions for how to disambiguate.

# Drawbacks

As mentioned above, the main drawback I see here is the potential for introducing confusion when a dependency conflict
is flagged. If ember-cli-dependency-lint is included in the default blueprint, providing clear guidance on how to
resolve issues will be important.

# Alternatives

- Do nothing. Folks have survived the problem for this long, and those who are concerned can install the addon
  themselves.

- Bake dependency validation logic into Ember CLI itself. At present this nicely fits the "if it can be an addon, make
  it an addon" philosophy, but it's possible there could be benefits from deeper integration.

# Unresolved questions

- What's the best way to educate users on how to resolve conflicts? The
  [existing README](https://github.com/salsify/ember-cli-dependency-lint#dealing-with-conflicts)
  outlines a few options, but can they be streamlined, and how do we ensure they're seen at the right time?

- Should the addon blueprint also include this? My leaning is no, but I can see some value in making sure a particular
  addon and its dependencies "play ball" and don't introduce issues all on their own.
