- Start Date: 2017-4-23

# Summary

It should be possible to specify packages/addons in `optionalDependencies` of the `package.json` of an `ember-cli project`, and ember-cli should scan for packages/addons mentioned in optionalDependencies while processing the build so that such packages/addons could also be included into the consuming application.

The build need not fail asserting "missing dependency" if any of the dependencies specified in optionalDependencies is missing/absent.

# Motivation

In general, the current ember-cli build process will scan for the packages specified in the `dependencies` hash and `devDependencies hash` from the downloaded packages in the node_modules folder, discovers and then includes them into the consuming application. The build is designed to fail if any of the packages specified in these two dependencies hash is missing in the `node_modules` folder. But this procedure may not be sufficient for a variety of cases.

So there could be an option for the developer to specify packages in optionalDependencies and ember-cli can lookup optionalDependencies while processing the build. The Build need not fail if there is any package specified in optionalDependencies is missing, since it is only optional and moreover may only be required for developmental purposes. This way the developer can have more control over the choice of packages he wishes to use for development and skip for production by giving appropriate commands like `npm install --no-optional`, thereby preventing the installation of packages itself rather than blacklisting in `ember-cli-build.js` which suggests preventing the installed addons sepcifed in the `blacklist` array from being included into the consuming application.

# Detailed design
We can tweak ember-cli addon/package discovery process to lookup for optionalDependencies as well and if the package is missing, we can make ember-cli proceed the build without terminating.

# How We Teach This

This functionality can simply be documented in ember-cli guides to teach.

# Alternatives

None.
