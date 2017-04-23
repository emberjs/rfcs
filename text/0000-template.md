- Start Date: 2017-4-23

# Summary

It should be possible to specify addons in optionalDependencies in package.json of an ember-cli project, and the ember-cli should scan optionalDependencies for addons while processing the build so that this addon tree will also be merged with the consuming application's app tree.
The build need not fail asserting "missing dependency" if the dependency is specified in optionalDependencies.

# Motivation

For users who consume the node packages via private npm registry, it is a pain point to install the addons when some of it's nested dependencies is specified with a (https://..) repository URL rather than regular "name" : "version" format. Since the consuming npm registry is private, it might not be possible to make a hit to global npm registry overriding the proxy everytime, for the packages which will never actually used in production.
Somehow the developer can preinstall the package(so while running npm install, there will not be a network hit for this package since the package of the same version would already have been installed and available) on his device and use it for development to run tests etc., but with a cost of making an entry in dependencies/devDependencies of package.json of the project. But while taking it to production there is a manual intervention of removing this entry from package.json (otherwise the build is ought to fail, bcoz the package cannot be fetched by private registry resulting ember-cli to throw the assertion - 'missing dependecy') every time by everyone working on the project which is the real pain point.
So there could be an option to the developer to include the addons as optionalDependencies and convey ember-cli whether or not to lookup optionalDependencies while processing the build.
The Build need not fail if there is any package specified in optionalDependencies is missing, since it is only optional.
This way the developer could have more control over the choice of packages he wishes to use for development and skip for production by giving appropriate commands like npm install --no-optional, thereby preventing the installation of packages itself rather than blacklisting in ember-cli-build.js which suggests excluding the installed addons sepcifed in the blacklist to be merged with the consuming application's app tree.

# Detailed design
